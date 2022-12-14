---
title: Distributed System__Instant messaging system mode
date: 2022-09-12 23:39:00 +0400
categories: [Courses, Distributed System]
tags: [Go Instant messaging System Programming]
img_path: /assets/img/Courses/Distributed System
---
# Demo1: Instant messaging system - Server
[reference](https://www.yuque.com/aceld/mo95lb/ks1lr9)
## 1. Construct basic Server
server.go, contains following;
- Server Struct, containing Ip, Port
- NewServer(ip String, port int) method to create server object
- (s *Server) Start() Start Server method
- (s *Server) Handler(conn net.Conn) handle connection

```go
package main

import (
    "fmt"
    "net"
)

type Server struct {
    Ip   string
    Port int
}

// 创建一个server的接口
func NewServer(ip string, port int) *Server {
    server := &Server{
        Ip:   ip,
        Port: port,
    }
    return server
}

func (s *Server) Handler(conn net.Conn) {
    // 当前连接的业务
    fmt.Println("Connected Successfully!")
}

// 启动服务器的接口
func (s *Server) Start() {
    // socket listen
    listener, err := net.Listen("tcp", fmt.Sprintf("%s:%d", s.Ip, s.Port))
    if err != nil {
        fmt.Println("net.Listen err: ", err)
        return
    }
    // close listen socket
    defer listener.Close()

    for {
        // accpet
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println("listener accept err: ", err)
            continue
        }
        // do handler
        go s.Handler(conn)
    }

}
```
main.go , start the Server:
```go
package main

func main() {
    server := NewServer("127.0.0.1", 8888)
    server.Start()
}
```

```dotnetcli
Linux or MacOS command:
build these two file: go build -o server main.go server.go
run: ./server
listen to the service: nc 127.0.0.1 8888
```

## 2. User online function
![1](1.png)

user.go:
- NewUser(conn net.Conn) *User 创建一个 user 对象
- (u *User) ListenMessage() 监听 user 对应的 channel 消息
```go
type User struct {
    Name string
    Addr string
    C    chan string
    conn net.Conn
}

// 创建一个用户的API
func NewUser(conn net.Conn) *User {
    // from server accept, remote address is user's address itself
    userAddr := conn.RemoteAddr().String()

    user := &User{
        Name: userAddr,
        Addr: userAddr,
        C:    make(chan string),
        conn: conn,
    }

    // 启动监听当前user channel消息的goroutine
    go user.ListenMessage()

    return user
}

// 监听当前user channel的方法，一旦有消息，直接发送给客户端
func (u *User) ListenMessage() {
    for {
        msg := <-u.C
        u.conn.Write([]byte(msg + "\n"))
    }
}
```

server.go:
- Add OnlineMap and Message fields
- Create and add new user when handling client connection
- Add listen and broadcast channel method
- use a goroutine to listen Message

```go
/*
 * @Author: Zeping Zhu
 * @Andrew ID: zepingz
 * @Date: 2022-09-06 03:00:10
 * @LastEditTime: 2022-09-11 23:03:11
 * @LastEditors: Zeping Zhu
 * @Description:
 * @FilePath: /learnGo/14_golang_IM_System/server.go
 */
package main

import (
	"fmt"
	"net"
	"sync"
)

type Server struct {
	Ip   string
	Port int

	// online user's map
	OnlineMap map[string]*User
	mapLock   sync.RWMutex

	// broadcast channel
	Message chan string
}

// A server object
func NewServer(ip string, port int) *Server {
	// var server *Server
	server := &Server{
		Ip:        ip,
		Port:      port,
		OnlineMap: make(map[string]*User),
		Message:   make(chan string),
	}
	return server
}

// goroutine to listen the message channel to send to all online user
func (s *Server) ListenMessage() {
	for {
		msg := <-s.Message
		// send msg to all online user
		s.mapLock.Lock()
		for _, cli := range s.OnlineMap {
			cli.C <- msg
		}
		s.mapLock.Unlock()
	}

}

// method to broadcast msgs
func (s *Server) Broadcast(user *User, msg string) {
	sendMsg := "[" + user.Addr + "]" + user.Name + ":" + msg
	s.Message <- sendMsg
}

func (s *Server) Handler(conn net.Conn) {
	// ... current connect business
	fmt.Println("Connect successfully!")
	user := NewUser(conn)
	// user online, add user to onlineMap
	s.mapLock.Lock()
	s.OnlineMap[user.Name] = user
	s.mapLock.Unlock()
	// broadcast current user's online msg
	s.Broadcast(user, "Online!")
	// handler block
	select {}
}

func (s *Server) Start() {

	// socket listen
	Listener, err := net.Listen("tcp", fmt.Sprintf("%s:%d", s.Ip, s.Port))
	if err != nil {
		fmt.Println("net.Listen err:", err)
		return
	}
	// close listen socket
	defer Listener.Close()

	// Start listen Message goroutine
	go s.ListenMessage()
	for {
		// accept
		conn, err := Listener.Accept()
		if err != nil {
			fmt.Println("net.Listen err:", err)
			continue
		}
		// dp handler
		go s.Handler(conn)
	}
}
```
```dotnetcli
结构体中的channel基本都需要开一个循环去监听其变化（尝试取出值，发送给其他channel）
```

## 3. Broadcast clients' messages
server.go: Improve the handle processing business method and start a read routine for the current client
```go
/*
 * @Author: Zeping Zhu
 * @Andrew ID: zepingz
 * @Date: 2022-09-06 03:00:10
 * @LastEditTime: 2022-09-11 23:18:59
 * @LastEditors: Zeping Zhu
 * @Description:
 * @FilePath: /learnGo/14_golang_IM_System/server.go
 */
package main

import (
	"fmt"
	"io"
	"net"
	"sync"
)

type Server struct {
	Ip   string
	Port int

	// online user's map
	OnlineMap map[string]*User
	mapLock   sync.RWMutex

	// broadcast channel
	Message chan string
}

// A server object
func NewServer(ip string, port int) *Server {
	// var server *Server
	server := &Server{
		Ip:        ip,
		Port:      port,
		OnlineMap: make(map[string]*User),
		Message:   make(chan string),
	}
	return server
}

// goroutine to listen the message channel to send to all online user
func (s *Server) ListenMessage() {
	for {
		msg := <-s.Message
		// send msg to all online user
		s.mapLock.Lock()
		for _, cli := range s.OnlineMap {
			cli.C <- msg
		}
		s.mapLock.Unlock()
	}

}

// method to broadcast msgs
func (s *Server) Broadcast(user *User, msg string) {
	sendMsg := "[" + user.Addr + "]" + user.Name + ":" + msg
	s.Message <- sendMsg
}

func (s *Server) Handler(conn net.Conn) {
	// ... current connect business
	fmt.Println("Connect successfully!")
	user := NewUser(conn)

	// user online, add user to onlineMap
	s.mapLock.Lock()
	s.OnlineMap[user.Name] = user
	s.mapLock.Unlock()

	// broadcast current user's online msg
	s.Broadcast(user, "Online!")

	// Receiver users' msgs
	go func() {
		buf := make([]byte, 4096)
		for {
			n, err := conn.Read(buf)
			if n == 0 {
				s.Broadcast(user, "Offline")
				// ctrl + c: will kill the user pid, and broadcast offline to other users
				// return to end the routine, every user has one!
				return
			}
			if err != nil && err != io.EOF {
				fmt.Println("Conn Read err:", err)
				return
			}
			// extract msg (delete '\n' in the end)
			msg := string(buf[:n-1])

			// broadcast msg
			s.Broadcast(user, msg)
		}
	}()

	// handler block
	select {}
}

func (s *Server) Start() {

	// socket listen
	Listener, err := net.Listen("tcp", fmt.Sprintf("%s:%d", s.Ip, s.Port))
	if err != nil {
		fmt.Println("net.Listen err:", err)
		return
	}
	// close listen socket
	defer Listener.Close()

	// Start listen Message goroutine
	go s.ListenMessage()
	for {
		// accept
		conn, err := Listener.Accept()
		if err != nil {
			fmt.Println("net.Listen err:", err)
			continue
		}
		// dp handler
		go s.Handler(conn)
	}
}
```
## 4. Encapsulate user API
user.go:
- Add server field in user struct
- Add Online, Offline, DoMessage methods

```go
/*
 * @Author: Zeping Zhu
 * @Andrew ID: zepingz
 * @Date: 2022-09-11 22:47:44
 * @LastEditTime: 2022-09-11 23:35:04
 * @LastEditors: Zeping Zhu
 * @Description:
 * @FilePath: /learnGo/14_golang_IM_System/User.go
 */
package main

import "net"

type User struct {
	Name string
	Addr string
	C    chan string
	conn net.Conn

	server *Server
}

// 创建一个用户的API
func NewUser(conn net.Conn, server *Server) *User {
	userAddr := conn.RemoteAddr().String()

	user := &User{
		Name:   userAddr,
		Addr:   userAddr,
		C:      make(chan string),
		conn:   conn,
		server: server,
	}

	// 启动监听当前user channel消息的goroutine
	go user.ListenMessage()

	return user
}

// user online
func (u *User) Online() {
	// user online, add user to onlineMap
	u.server.mapLock.Lock()
	u.server.OnlineMap[u.Name] = u
	u.server.mapLock.Unlock()

	// broadcast online msg
	u.server.Broadcast(u, "Online!")
}

// user offline
func (u *User) Offline() {
	// user offline, delete it from onlineMap
	u.server.mapLock.Lock()
	delete(u.server.OnlineMap, u.Name)
	u.server.mapLock.Unlock()

	// broadcast offline msg
	u.server.Broadcast(u, "Offline!")
}

// handle user msg
func (u *User) DoMessage(msg string) {
	u.server.Broadcast(u, msg)
}

// 监听当前user channel的方法，一旦有消息，直接发送给客户端
func (u *User) ListenMessage() {
	for {
		msg := <-u.C
		u.conn.Write([]byte(msg + "\n"))
	}
}
```

server.go:
- Use user API to replace code before

```go
/*
 * @Author: Zeping Zhu
 * @Andrew ID: zepingz
 * @Date: 2022-09-06 03:00:10
 * @LastEditTime: 2022-09-11 23:36:35
 * @LastEditors: Zeping Zhu
 * @Description:
 * @FilePath: /learnGo/14_golang_IM_System/server.go
 */
package main

import (
	"fmt"
	"io"
	"net"
	"sync"
)

type Server struct {
	Ip   string
	Port int

	// online user's map
	OnlineMap map[string]*User
	mapLock   sync.RWMutex

	// broadcast channel
	Message chan string
}

// A server object
func NewServer(ip string, port int) *Server {
	// var server *Server
	server := &Server{
		Ip:        ip,
		Port:      port,
		OnlineMap: make(map[string]*User),
		Message:   make(chan string),
	}
	return server
}

// goroutine to listen the message channel to send to all online user
func (s *Server) ListenMessage() {
	for {
		msg := <-s.Message
		// send msg to all online user
		s.mapLock.Lock()
		for _, cli := range s.OnlineMap {
			cli.C <- msg
		}
		s.mapLock.Unlock()
	}

}

// method to broadcast msgs
func (s *Server) Broadcast(user *User, msg string) {
	sendMsg := "[" + user.Addr + "]" + user.Name + ":" + msg
	s.Message <- sendMsg
}

func (s *Server) Handler(conn net.Conn) {
	// ... current connect business
	fmt.Println("Connect successfully!")
	user := NewUser(conn, s)

	// user online, add user to onlineMap
	user.Online()

	// broadcast current user's online msg
	s.Broadcast(user, "Online!")

	// Receiver users' msgs
	go func() {
		buf := make([]byte, 4096)
		for {
			n, err := conn.Read(buf)
			if n == 0 {
				user.Offline()
				// ctrl + c: will kill the user pid, and broadcast offline to other users
				// return to end the routine, every user has one!
				return
			}
			if err != nil && err != io.EOF {
				fmt.Println("Conn Read err:", err)
				return
			}
			// extract msg (delete '\n' in the end)
			msg := string(buf[:n-1])

			// broadcast msg
			user.DoMessage(msg)
		}
	}()

	// handler block
	select {}
}

func (s *Server) Start() {

	// socket listen
	Listener, err := net.Listen("tcp", fmt.Sprintf("%s:%d", s.Ip, s.Port))
	if err != nil {
		fmt.Println("net.Listen err:", err)
		return
	}
	// close listen socket
	defer Listener.Close()

	// Start listen Message goroutine
	go s.ListenMessage()
	for {
		// accept
		conn, err := Listener.Accept()
		if err != nil {
			fmt.Println("net.Listen err:", err)
			continue
		}
		// dp handler
		go s.Handler(conn)
	}
}
```

## 5. Search online users and rename userName
### 5.1 Search online users
if user inputs 'who', then return online users to him
user.go:
- Add send message API
```go
func (u *User) SendMsg(msg string) {
    u.conn.Write([]byte(msg))
}
```
- In DoMessage() method, add function to handle "who" msg, output online users' info
```go
func (u *User) DoMessage(msg string) {
	if msg == "who" {
		// look for current online users
		u.server.mapLock.Lock()
		for _, user := range u.server.OnlineMap {
			onlineMsg := "[" + user.Addr + "]" + user.Name + ":" + "Online...\n"
			u.SendMsg(onlineMsg)
		}
		u.server.mapLock.Unlock()
	} else {
		u.server.Broadcast(u, msg)
	}
}
```
### 5.2 Rename userName
if a user inputs "rename|zzp", it will change his name to "zzp"
user.go:
- In DoMessage() method, add case for handling "rename|xxx"
```go
func (u *User) DoMessage(msg string) {
	if msg == "who" {
		// look for current online users
		u.server.mapLock.Lock()
		for _, user := range u.server.OnlineMap {
			onlineMsg := "[" + user.Addr + "]" + user.Name + ":" + "Online...\n"
			u.SendMsg(onlineMsg)
		}
		u.server.mapLock.Unlock()
	} else if len(msg) > 7 && msg[:7] == "rename|" {
		// format:"rename|xxx"
		newName := strings.Split(msg, "|")[1]
		// check if newName has existed
		_, ok := u.server.OnlineMap[newName]
		if ok {
			u.SendMsg("This userName has been used!\n")
		} else {
			u.server.mapLock.Lock()
			delete(u.server.OnlineMap, newName)
			u.server.OnlineMap[newName] = u
			u.server.mapLock.Unlock()

			u.Name = newName
			u.SendMsg("You have updated the userName: " + u.Name + "\n")
		}
	} else {
		u.server.Broadcast(u, msg)
	}
}
```

## 6. Close the user when exceeds time limit
When a user hasn't sent a msg for a long time, he will be kicked.

server.go:
- In handler() goroutine, add islive channel, if the user send a message, add true
```go
func (s *Server) Handler(conn net.Conn) {
	// ... current connect business
	fmt.Println("Connect successfully!")
	user := NewUser(conn, s)

	// user online, add user to onlineMap
	user.Online()

	//check whether user is alive
	islive := make(chan bool)

	// Receiver users' msgs
	go func() {
		buf := make([]byte, 4096)
		for {
			n, err := conn.Read(buf)
			if n == 0 {
				user.Offline()
				// ctrl + c: will kill the user pid, and broadcast offline to other users
				// return to end the routine, every user has one!
				return
			}
			if err != nil && err != io.EOF {
				fmt.Println("Conn Read err:", err)
				return
			}
			// extract msg (delete '\n' in the end)
			msg := string(buf[:n-1])

			// broadcast msg
			user.DoMessage(msg)

			islive <- true
		}
	}()

	for {
		// select理解： 首先执行case右边语句并评估，都没有可行的就会阻塞，超过一个可行会随机选中case
		// 然后进入下一次轮询，每次轮询都会重新执行所有case右边语句并开始评估！
        // 参考：https://www.cnblogs.com/f-ck-need-u/p/9994512.html
		select {
		// if islive, do nothing, and select will enter into next check,
		// also the time counting will be reset.
		// 把time.After() 写在里面的话，如果islive case选中，下一次轮询就会重置计时
		case <-islive:
			// current user is alive, do nothing and reset time in next check
		case <-time.After(time.Second * 10):
			// time exceeds! kick the user
			user.SendMsg("You are kicked!\n")

			// close resources
			close(user.C)
			conn.Close()

			// exit
			return
		}
	}
	// handler block
	select {}
}
```

## 7. Private communication mode
msg format: to|name|content
will send message to a specific user
```go
func (u *User) DoMessage(msg string) {
	if msg == "who" {
		// look for current online users
		u.server.mapLock.Lock()
		for _, user := range u.server.OnlineMap {
			onlineMsg := "[" + user.Addr + "]" + user.Name + ":" + "Online...\n"
			u.SendMsg(onlineMsg)
		}
		u.server.mapLock.Unlock()
	} else if len(msg) > 7 && msg[:7] == "rename|" {
		// format:"rename|xxx"
		newName := strings.Split(msg, "|")[1]
		// check if newName has existed
		_, ok := u.server.OnlineMap[newName]
		if ok {
			u.SendMsg("This userName has been used!\n")
		} else {
			u.server.mapLock.Lock()
			delete(u.server.OnlineMap, newName)
			u.server.OnlineMap[newName] = u
			u.server.mapLock.Unlock()

			u.Name = newName
			u.SendMsg("You have updated the userName: " + u.Name + "\n")
		}
	} else if len(msg) > 4 && msg[:3] == "to|" {
		// msg format: to|zzp|content
		// 1. get receiver's username
		remoteName := strings.Split(msg, "|")[1]
		if remoteName == "" {
			u.SendMsg("Please give a valid name! format is to|name|content\n")
			return
		}
		// 2. According to the username, get the user object
		remoteUser, ok := u.server.OnlineMap[remoteName]
		if !ok {
			u.SendMsg("This user doesn't exist!\n")
			return
		}

		// 3. remoteUser exists, send msg to remoteUser
		content := strings.Split(msg, "|")[2]
		if content == "" {
			u.SendMsg("Content is not valid\n")
			return
		} else {
			remoteUser.SendMsg(u.Name + " says:" + content)
		}

	} else {
		u.server.Broadcast(u, msg)
	}
}
```

# Demo2: Instant messaging system - Client
## Define client struct and connection
client.go:
```go
/*
 * @Author: Zeping Zhu
 * @Andrew ID: zepingz
 * @Date: 2022-09-13 00:05:29
 * @LastEditTime: 2022-09-13 00:12:57
 * @LastEditors: Zeping Zhu
 * @Description:
 * @FilePath: /learnGo/14_golang_IM_System/client.go
 */
package main

import (
	"fmt"
	"net"
)

type Client struct {
	ServerIp   string
	ServerPort int
	Name       string
	conn       net.Conn
}

func NewClient(serverIp string, serverPort int) *Client {
	// create a new client object
	client := &Client{
		ServerIp:   serverIp,
		ServerPort: serverPort,
	}
	// connect to server
	conn, err := net.Dial("tcp", fmt.Sprintf("%s:%d", serverIp, serverPort))
	if err != nil {
		fmt.Println("net.Dial error: ", err)
		return nil
	}
	client.conn = conn
	// return object
	fmt.Println("Connect successfully! localaddr:")
	fmt.Println(conn.LocalAddr())
	fmt.Println("Connect successfully! Remoteaddr:")
	fmt.Println(conn.RemoteAddr())
	return client
}
func main() {
	client := NewClient("127.0.0.1", 8888)
	if client == nil {
		fmt.Println(">>> connect fail!")
		return
	}
	fmt.Println(">>> Connect succeeds!")
	select {}
}
```
```dotnetcli
go build -o client client.go
run: ./client
```

## Parse command line
Initialize command line args and parse them in init function
```go
var serverIp string
var serverPort int

func init() {
	flag.StringVar(&serverIp, "ip", "127.0.0.1", "Set server ip addr(default: 127.0.0.1)")
	flag.IntVar(&serverPort, "port", 8888, "Set server port(default: 8888)")

	// parse command line
	flag.Parse()
}
```

in main function:
```go
client := NewClient(serverIp, serverPort)
```
Can parse args in command line to run client
```dotnetcli
./client -h : get help
./client -ip 127.0.0.1 -port 8888
```

## Menu display
add new flag field in client struct:
```go
type Client struct {
	ServerIp   string
	ServerPort int
	Name       string
	conn       net.Conn
	flag       int // current client mode
}
```
add menu function to get input
```go
func (client *Client) menu() bool {
	var flag int
	fmt.Println("1. Public messaging mode")
	fmt.Println("2. Private messaging mode")
	fmt.Println("3. Update userName")
	fmt.Println("0. Exit")
	fmt.Scanln(&flag)
	if flag >= 0 && flag <= 3 {
		client.flag = flag
		return true
	} else {
		fmt.Println(">>> Please enter valid number (0 ~ 3)")
		return false
	}
}
```

add run() to loop main business:
```go
func (client *Client) Run() {
	for client.flag != 0 {
		for !client.menu() {
		}
		// Handle different business
		switch client.flag {
		case 1:
			// Public
			fmt.Println("Public")
			break
		case 2:
			// Private
			fmt.Println("Private")
		case 3:
			// Update user name
			fmt.Println("Update user name")
		}
	}
	fmt.Println("Exit!")
}
```
add flag in new client function:
```go
func NewClient(serverIp string, serverPort int) *Client {
	// create a new client object
	client := &Client{
		ServerIp:   serverIp,
		ServerPort: serverPort,
		flag:       999, // if 0, will exit directly!
	}
	// connect to server
	conn, err := net.Dial("tcp", fmt.Sprintf("%s:%d", serverIp, serverPort))
	if err != nil {
		fmt.Println("net.Dial error: ", err)
		return nil
	}
	client.conn = conn
	// return object
	fmt.Println("Connect successfully! localaddr:")
	fmt.Println(conn.LocalAddr())
	fmt.Println("Connect successfully! Remoteaddr:")
	fmt.Println(conn.RemoteAddr())
	return client
}
```

## Update user name
Add UpdateName() function to update user name:
```go
func (client *Client) UpdateName() bool {
	fmt.Println(">>>Please input username: ")
	fmt.Scanln(&client.Name)
	sendMsg := "rename|" + client.Name + "\n"    // encapsulate protocal
	_, err := client.conn.Write([]byte(sendMsg)) // server will read the msg and handle it
	if err != nil {
		fmt.Println("conn.Write err: ", err)
		return false
	}
	return true
}
```

Add DealResponse() method to handle server's response msg
Reason: the run() method will always block for input, need another goroutine to handle server's response
```go
func (client *Client) DealResponse() {
	io.Copy(os.Stdout, client.conn)
	// be like following: Once client.conn has data can read, copy to stdout, always block and listen!
	// for {
	// 	buf := make()
	// 	client.conn.Read(buf)
	// 	fmt.Println("")
	// }
}
```

Open a goroutine in main, to deal with response!
```go
func main() {
	client := NewClient(serverIp, serverPort)
	if client == nil {
		fmt.Println(">>> connect fail!")
		return
	}
	// open a goroutine to handle server's feedback msg
	go client.DealResponse()
	fmt.Println(">>> Connect succeeds!")
	client.Run()
	select {}
}
```

## Public messaging mode
Add PublicChat():
```go
func (client *Client) PublicChat() {
	// user input msg
	var chatMsg string

	fmt.Println(">>> Please enter msg, input exit to exit")
	fmt.Scanln(&chatMsg)
	for chatMsg != "exit" {
		// send to server
		// msg != null, send at once
		if len(chatMsg) != 0 {
			sendMsg := chatMsg + "\n"
			_, err := client.conn.Write([]byte(sendMsg))
			if err != nil {
				fmt.Println("conn Write err: ", err)
				break
			}
		}
		chatMsg = ""
		fmt.Println(">>> Please enter msg, input exit to exit")
		fmt.Scanln(&chatMsg)
	}
}
```

## Private chatting mode
Search online clients:
```go
func (client *Client) SelectUsers() {
	sendMsg := "who\n"
	_, err := client.conn.Write([]byte(sendMsg))
	if err != nil {
		fmt.Println("conn Write err: ", err)
		return
	}
}
```

Add private mode:
```go
func (client *Client) PrivateChat() {
	var remoteName string
	var chatMsg string

	client.SelectUsers()
	fmt.Println(">>>>请输入聊天对象的[用户名], exit退出: ")
	fmt.Scanln(&remoteName)

	for remoteName != "exit" {
		fmt.Println(">>>>请输入消息内容，exit退出:")
		fmt.Scanln(&chatMsg)

		for chatMsg != "exit" {
			// 消息不为空则发送
			if len(chatMsg) != 0 {
				sendMsg := "to|" + remoteName + "|" + chatMsg + "\n\n"
				_, err := client.conn.Write([]byte(sendMsg))
				if err != nil {
					fmt.Println("conn Write err: ", err)
					break
				}
			}
			chatMsg = ""
			fmt.Println(">>>>请输入消息内容，exit退出:")
			fmt.Scanln(&chatMsg)
		}

		client.SelectUsers()
		fmt.Println(">>>>请输入聊天对象的[用户名], exit退出: ")
		fmt.Scanln(&remoteName)

	}

}
```