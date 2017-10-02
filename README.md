# Instachat

Elixir and Phoenix. 

Phoenix is a web framework that built in Elixir and runs on Erlang VM.

Erlang VM: Erlang VM was invented in 1986 for phone switches. It is a general-purpose programming language and runtime environment. Runtime environment means that the code is compiled, runs everywhere, and can be update without crashing anything. It works with functional programing. 

What is functioning programming you may ask:
Type of computer programming that avoids changing states and mutable data. It uses pure functions. By using functional programming the same input will always give the same output so functions are independent from other functions. This is unlike object oriented programming which shares states in objects and methods so you could get different outputs. States can be changed with object oriented programming.

Okay back to Elixir and Phoenix - What is Elixir?
José Valim, who worked with Rails, saw potential in Erlang and its productivity but wanted to modernize it. Elixir is thus a functional language that is built on Erlang. You can think about it like the modern version of Erlang. It’s syntax is similar to Ruby. 

Ruby + Rails vs Elixir + Phoenix
Looks like Ruby and Rails are very similar but Ruby/Rails is a bit more popular. One major difference is Ruby/Rails is object oriented whereas Elixir/Phoenix is functional. Since Erlang has such great productivity and speed, Elixir/Phoenix is faster.

What is it especially good for?
It is used for websites. It is particular good for chat and voice applications. 
Example companies that use it: WhatsApp and Pinterest

------

Instachat

The Phoenix Framework was built with realtime communication as a first class priority. Using its built in socket handling and channels we can implement a basic, realtime chat application with little effort.

For this video we’re going to assume that you already have Elixir and Phoenix Setup. You will not need a database as the messages will not be persisted. This tutorial is taken pretty much directly from the Phoenix Documentation.

Setting up the app

To start let’s generate a standard phoenix application:

$> mix phoenix.new instachat
And get it running:

$> cd instachat
$> mix phoenix.server
Now in a web browser hitting http://localhost:4000 should give us the phoenix start page.

Setting Up the Socket

When we ran $> mix phoenix.new it created a default socket module for us and attached it to the url /socket. Let's open up lib/instachat/endpoint.ex and check it out:

# in file: lib/instachat/endpoint.ex  

socket "/socket", Instachat.UserSocket
This is telling Phoenix that all socket connections hitting /socket should be handled by the Instachat.UserSocket module. This UserSocket module is where we handle all the configuration for the socket itself like connecting and routing messages. It lives at web/channels/user_socket.ex. Let's open it up and have a look.

Up at the top we see some commented out code referencing channels:

# in file: web/channels/user_socket.ex

## Channels
  # channel "rooms:*", Instachat.RoomChannel
The channel "rooms:*", Instachat.RoomChannel line is boiler plate example code for handling messages coming over this socket. It says, send any messages that come in starting with "rooms:" and ending with anything to the Instachat.RoomChannel module. This is good enough for our purposes so let's uncomment that line:

# in file: web/channels/user_socket.ex

## Channels
  channel "rooms:*", Instachat.RoomChannel
Setting up the Channel

The channel module wasn't created for us automatically so let's create it ourselves. It is going to live at web/channels/room_channel.ex and here's the boilerplate:

#in file: web/channels/room_channel.ex

defmodule Instachat.RoomChannel do
  use Phoenix.Channel
end
The first thing a channel needs to do is handle connections. We do this by implementing a function called join that either returns {:ok, socket} on a successful join or {:error, message} otherwise. Let's write code that lets users join only if they try to join the lobby, otherwise we'll deny them:

#in file: web/channels/room_channel.ex

defmodule Instachat.RoomChannel do
  use Phoenix.Channel
  def join("rooms:lobby", _message, socket) do
    {:ok, socket}
  end
  def join(_room, _params, _socket) do
    {:error, %{reason: "you can only join the lobby"}}
  end
end
Connecting From Javascript

The boilerplate javascript for connecting to our socket from a web browser has already been written for us but is not being loaded by default. If we open up web/static/js/app.js and look down at the bottom we can see that the code to do this is commented out. Let's un-comment that line:

//in file: web/static/js/app.js

import socket from "./socket"
Now with our web browser pointed to http://localhost:4000/ and the developer console open we can see the message:

Unable to join Object {reason: "unmatched topic"}
This is because our javascript is trying to connect to our socket over a topic that we aren't handling. Let's open up the javascript and set it to the right topic. This javascript file lives at web/static/js/socket.js and the code in concern is down at the bottom:

//in file: web/static/js/socket.js

// Now that you are connected, you can join channels with a topic:
let channel = socket.channel("topic:subtopic", {})
This code is trying to connect to a channel called "topic" with a sub-topic of "subtopic" but we want to connect to "rooms:lobby" Let's go ahead and change that:

//in file: web/static/js/socket.js

// Now that you are connected, you can join channels with a topic:
let channel = socket.channel("rooms:lobby", {})
And now if we check in our browser's console we should see:

Joined successfully Object {}
This means that we've both connected to the Socket and Joined the Channel

Adding the HTML

To interact with the chat we're going to need some user interface. Let's add places to input and display messages. Open up web/templates/page/index.html.eex and replace its entire contents with:

<!-- in file: web/templates/page/index.html.eex -->

<div id="messages"></div>
<input id="chat-input" type="text"></input>
Hooking up the HTML

For this demo we're gonna keep it simple and use jQuery. Let's add a CDNd version to the application layout which is located at web/templates/layout/app.html.eex right above the application js file:

<!-- in file: web/templates/layout/app.html.eex -->

</div> <!-- /container -->
    <!-- add the following line -->
    <script src="//code.jquery.com/jquery-1.11.3.min.js"></script>
    <script src="<%= static_path(@conn, "/js/app.js") %>"></script>
  </body>
Back in the javascript file for working with this socket let's add some code to hook up the HTML we just added. Down at the bottom of the file add:

// in file: web/static/js/socket.js

// UI Stuff
let chatInput = $("#chat-input");
let messagesContainer = $("#messages");

chatInput.on("keypress", event => {
  if(event.keyCode === 13){
    channel.push("new_message", {body:chatInput.val()});
    chatInput.val("");
  }
});
All this code does is call the push method on channel when we press the enter key. It gives push two arguments, an event name of "new_message" and a payload which is an object containing our message. Channel is going to send this back to our phoenix app. So let's handle it.

Handling Channel Events

Back in our RoomChannel module we need to handle events coming in and broadcast them to all our connected clients. All we have to do is implement a handle_in function. Let's add it below our join functions:

# in file: web/channels/room_channel.ex

  def handle_in("new_message", body, socket) do
    broadcast! socket, "new_message", body
    {:noreply, socket}
  end
We can see that we're pattern matching on events with the name of "new_message", then we simply broadcast the message out to all our connected clients, and we return {:noreply, socket} which is one of the required return values of handle_in and means that the client that sent the message doesn't get anything back from our channel directly. Now we need to receive the broadcast from our Javascript and display the message.

Receiving Events in Javascript

Back in our Javascript file we need to look out for our "new_message" event and update the messages display when we get one. Down at the bottom of web/static/js/socket.js lets add:

// in file: web/static/js/socket.js

channel.on("new_message", payload => {
  messagesContainer.append(`<br/>[${Date()}] ${payload.body}`)
})
This code simply tells the channel to look out for events named "new_message" and to run a function that adds the payload's body to the messages container when we get one. That's it, we should be all done! Let's open up the browser and give it a try.

Testing

Pointing our browser to http://localhost:4000/ , typing something into the input, and pressing enter we should now see the chat working. If we open up another tab we should be able to see any new messages in both tabs and in fact any connected web browsers should be able to see any new messages!

Wrapping it up

Phoenix makes it almost dead simple to write realtime applications for the modern web. With sockets we can handle routing of clients to channels and with channels we can handle receiving and broadcasting events to and from clients with ease. And we get to write this all with the power and clarity of Elixir!

This github takes directly from Instachat tutorial: https://gist.github.com/yaycode/58ff8213ea54d7272ae89d0b9165be16
Using their tutorial we set up a real time chat application with Elixir and Phoenix. 

------

Reading material and sources: 
https://hackernoon.com/phoenix-is-better-but-rails-is-more-popular-8975d5e68879
http://phoenixframework.org/
https://elixir-lang.org/getting-started/introduction.html
https://blog.carbonfive.com/2016/04/19/elixir-and-phoenix-the-future-of-web-apis-and-apps/


## Learn more

  * Official website: http://www.phoenixframework.org/
  * Guides: http://phoenixframework.org/docs/overview
  * Docs: https://hexdocs.pm/phoenix
  * Mailing list: http://groups.google.com/group/phoenix-talk
  * Source: https://github.com/phoenixframework/phoenix
