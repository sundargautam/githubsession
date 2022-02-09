
## Documentation

1. Basic Setup for Socket.io.

```

import { createServer } from "http";
import { Server, Socket } from "socket.io";

const app = express();
const server = createServer(app);
export const io = new Server(server, {
  cors: { 
    origin: "*",
  },
});

server.listen(PORT, () => {
 
});


```
Queries: Why to make a new instance of server when we already have 
an express server? In above case *app* is already there so why to 
create *server* again?

Ans: If you want socket.io to run on the same port
as your webserver, then you use same server instance.Thus, in the above
example i haved attached socket server to same port as express server
.


Now, let's see how to detect connection and disconnection of client
using socket.

```

import { createServer } from "http";
import { Server, Socket } from "socket.io";

const app = express();
const server = createServer(app);
export const io = new Server(server, {
  cors: { 
    origin: "*",
  },
});

server.listen(PORT, () => {
 
});

//this is listening to connection event.This will be triggered 
when client gets connected to this server socket.I will show
you demo later.Here, client is of type Socket, this is use for triggering
or listening event to/from client.


io.on('connection',(client:Socket)=>{

      //triggered when connection is established or when 
      client first connect to the server.
})



You might be wondering how to detect disconnection of client:

io.on('connection',(client:Socket)=>{

      //triggered when connection is established or when 
      client first connect to the server.

      client.on("disconnect",()=>{
        //triggers when client got disconnected
      })
})


```

In this above example, you might be wondering where to use this?

If you are thinking about developing an application, that will detect
whether user is online or offline, then you can use this approach
very effectivly.

Demo example:

```

io.on('connection',async(client: Socket): Promise<void> => {
    const { userId } = client.handshake.query; // we will learn this later.For Now just think
    someone we have passed userId .

    console.log("connected");
    console.log(userId);
    try {
      const userExist = await User.findById(userId);
      if (!userExist) {
        throw new Error("user doesn't exist");
      }
      await User.findByIdAndUpdate(
        userId,
        {
          isOnline: true,
        },
        {
          new: true,
        },
      );
    } catch (error) {
      console.log(error);
    }

    /* checking online till here */

    client.on("disconnect", async () => {
      try {
        const userExist = await User.findById(userId);
        if (!userExist) {
          throw new Error("user doesn't exist");
        }
        await User.findByIdAndUpdate(
          userId,
          {
            isOnline: false,
          },
          {
            new: true,
          },
        );
      } catch (error) {
        console.log(error);
      }
    });

    /* checking offline till here */)
```

This seems interesting right???


Till here we learnt how to write server side code for connection/disconnection
of client in socket.

Let's implement code in client side too.
In order to make a connection to this severside socket, we need client
socket library to make a connection to server.

Here, i will just embedded main code for execution. It depends upon
you how you will use this:

```

import socketIOClient from "socket.io-client";

//this is will create a socket instance at client side and
as soon as apiUrl is configured for socket, then this will
execute on connection portion code in the server side.

  const socket = socketIOClient(config.app.apiUrl);

```

Till here we learnt about how to create a basic socket in both client and server side.And, now we will be learning about creating events and listening events which is what we need.


## Custom/Broadcasting Event

Here, I want client to send message to server and server will display message sent by client.

```
io.on('connection',(client:Socket)=>{
  
  socket.on("client::message",(message)=>{
  //message is message sent by client
  
  
  });
}
);
```

Note: here , .on("event_name") is used to listen for the event and .emit("event_name") is used
for emiting to the event.
Thus, if you want to make somone either client/server to listen to event, at first you 
need to emit event and then corresponding targeted client/server will listen for that event.

Eg:
```
  const socket = socketIOClient(config.app.apiUrl);
  socket.emit("client::message","hello"); //here client is emiting event with message hello.
  Thus if someone wants to grab this message, then they should listen to this event "client::message".
  
  ## Server Side
  io.on('connection',(client:Socket)=>{
  
  socket.on("client::message",(message)=>{
  //message is message sent by client ,
  });
}
);
```

Using this approach you can create events based upon your requirement and perform taskes in real 
time.

Now, above I have included broadcasting right?? So, what is this??

Suppose there is a chat room and already 2 people have joint in that chat.Thus,if one new user
joint in that chat then expect him all other need to get message that new user have been joint.Here, expect for a user triggering event other user's are notified with message.


```
  ## ClientSide code
  const socket = socketIOClient(config.app.apiUrl);
  socket.emit("client::message","hello");
  
  
   ## Server Side
  io.on('connection',(client:Socket)=>{
  
  socket.on("client::message",(message)=>{
    socket.broadcast.emit("expect::current","expect current");
    
    //when you will listen to "expect::current", then client emiting "client::message" won't get message
  });
}
);
```
To make this broadcasting more relatable and understandable, we need to first learn *join* event in socket.

# What is Join?
Have you ever thought we can create our own custom room using socket and only that room memeber
will be able to receive messages? Yes, this is what we are going to achieve using join.

This is pretty easy to achiev, just like previous example we are going to achieve this:
Let's take an example and I guess this will help you to grab this concept much easily:

```
const bidingHistory = []; //this simulates our biding database

const room = ["phone", "tv"]; //avilable rooms

io.on("connection", (socket) => {
  console.log(socket.handshake.query.id, socket.handshake.query.name); //with this id i can change status of user in database
  socket.on("makeBid", (params) => {
    const { productId, auctionId, bidAmount, room, name, id } = params;

    bidingHistory.push({ ...params });
    console.log(bidingHistory);
    if (bidAmount >= Math.max(...bidingHistory.map((o) => o.bidAmount))) {
      socket.broadcast
        .to(room)
        .emit("bidMessage", "someone has bidded more than you");
    } else {
      socket.to(room).emit("bidMessage", `${name} participated in bidding`);
    }
  });

  socket.on("joinPhone", () => {
    console.log("triggered phone auction");
    socket.join(room[0]);
    socket.broadcast.to(room[0]).emit("success phone", "new member on phone");
  });

  socket.on("joinTv", () => {
    console.log("triggered tv auction");
    socket.join(room[1]);
    socket.to(room[0]).emit("success tv", "new member on tv");
  });

  socket.on("disconnect", () => {
    console.log(
      socket.handshake.query.id,
      socket.handshake.query.name,
      "disconnected"
    ); //with this id i can change status of user in database
  });
});
```

Above code explanation:
First, user need to join the group for Eg: either phone or tv group
```
  socket.on("joinPhone", () => {
    console.log("triggered phone auction");
    socket.join(room[0]);
    socket.broadcast.to(room[0]).emit("success phone", "new member on phone");
  });

  socket.on("joinTv", () => {
    console.log("triggered tv auction");
    socket.join(room[1]);
    socket.to(room[0]).emit("success tv", "new member on tv");
  });
```
Here, **Socket.join(groupname)** will allow user to joing their respective group and **socket.to(groupname).emi("same as before")** will let only the specific group clients to listen to that message. 
For eg: if u1,u2 has joined "phone" group and u3 joined "tv" group then only u1 and u2 will be able to listen on "success phone" event but u3 couldn't listen to this.

Seems interesting right??
If you further want to explore this have a look at this github repository:
[Github link for simulating above example](https://github.com/sundargautam/socket-nodejs/blob/master/server/server.js)









