#Bulding a chat forum on hydna#

> Some conversations are better in closed rooms.

##Forum##

![Forum running on Iphone](687474703a2f2f7777772e736b61676769766172612e636f6d2f64726f702f666f72756d5f6970686f6e652e706e67.png)

###What is it?###
Forum is an HTML and Javascript chat app with rooms. It’s all static assets and some hydna [behaviors](http://hydna.com/documentation/behaviors/introduction/), just create a [hydna account](https://www.hydna.com/account/signup/), upload the behavior files and you are in business. It also works on IOS devices.

##Setup##

1. Create an account on: [hydna.com](http://hydna.com/).
2. Enter your resulting details in **forum.js** (**ROOT_URL**).
3. Upload the behavior files and start chatting.

[Try it out](http://hydna.github.com/forum) or [Check out the source](http://www.github.com/hydna/forum)

###Room creation###

What makes the forum app more than just the most basic chat app, is the feature to create rooms. To do this we use behaviors. You can find ours in the behaviors folder.

####behaviors####
- setup.be
- onhandshake.js
- onjoinroom.js
- onleaveroom.js
- api_get_user_list.js
- api_get_rooms.js
- api_create_rooms.js

In **setup.be** we create our routing to different scripts. In this demo we have a maximum of 40 rooms with 50 users each, this can of course be changed with the appropriate hydna account settings.

	// setup.be

    LOBBY_CHANNEL = 1
    ROOM_OFFSET = 2
    MAX_ROOMS = 40
    MAX_USERS_PER_ROOM = 50
		
   	.... 


In this application we have LOBBY_CHANNEL and a channel for each room. If the user connects on the LOBBY_CHANNEL they are only allowed to emit signals, see the LOBBY_CHANNEL as a kind of entry point to create rooms or get a list of available rooms. In **forum.js** we have most of the client side hydna logic.

We start by connecting to the LOBBY_CHANNEL in emit mode only.
 
    // forum.js
	lobbyChannel = new HydnaChannel(ROOT_URL + "/" + LOBBY_CHANNEL, "e");

On the emit directive we look at the user provided token to see what they want to happen, in this case we have two options, **create_room** and **get_rooms** , these invoke **api_create_room.js** and **api_get_rooms.js** scripts respectively.
    
	// setup.be

	emit
		channel = LOBBY_CHANNEL
	
	    	token = match("^create_room")
	      		run("./api_create_room.js")
	    	end

	    	token = match("^get_rooms")
	      		run("./api_get_rooms.js")
	    	end
		end
	
	....

	// api_create_rooms.js
	
  	// Send back the channel id of the new room.
    message = "create_room:ok " + id;
    connection.reply(message);

    // Also send a notification to all other connected user. They
    // should be aware that the new room exists
    message = "notif:room-created " + [id, title].join(",");
    channel.emit(message);
	
  	....
    
  	// api_get_rooms.js
	
    result = [];

    for (var id = ROOM_OFFSET; id < MAX_ROOMS; id++) {
      room = domain.getChannel(id);
      if (room.get("active") == "yes") {
        str = [id, room.get("title"), room.get("count") || "0"].join(",");
        result.push(str);
      }
    }

    connection.reply("get_rooms:ok " + result.join(";"));	

  	....

Once we have a room list or created a room we can proceed to enter one.
    
	// forum.js
	
	var url = ROOT_URL + "/" + chanid + "?" + [userNick, userHash].join(",");
	var chan = new HydnaChannel(url, "rwe");
	
	....

On open on the LOBBY_CHANNEL we invoke **onhandshake.js** where we get the **connectionid** for the user and allow the connection. If the user is connecting to a room channel we try to allocate a place for them in the room channel cache. If the allocation is successful we notify all the other users in that room of their new friend, and allow the connection.
    
	// onjoinroom.js  
	
	// Try to allocate a slot in the room
	if (channel.incr("count", MAX_USERS_PER_ROOM) == false) {
	  exit(1, "ROOM_IS_FULL");
	}
	
	....
	
	// Add connection to the `connections` collection
	channel.push("connections", [connid, alias, image].join(","));

	// Tell everyone currently in room that a new
	// users has joined.
	message = "notif:user-join " + [connid, alias, image].join(",");
	channel.emit(message);

	// Tell lobby that a rooms info has been changed
	message = "notif:room-info " + [channel.id, channel.get("count")].join(",");
	domain.getChannel(LOBBY_CHANNEL).emit(message)


On close we remove the user from the room and also checks if this was the last user in this room we remove the room and notify on the LOBBY_CHANNEL what happened, if this is not the last user we notify the other users in the room what user just left.


	// onleaveroom.js

    // Convert connection to string
    connid = connection.id.toString(16);

    // Remove connection from channel colllection
    channel.rem("connections", new RegExp("^" + connid));
    channel.decr("count", 0);

    // Check if current connection was the last user in the room
    if (channel.get("count") == 0) {

      // Tell every user that room has been destroyed.
      message = "notif:room-destroyed " + channel.id;
      domain.getChannel(LOBBY_CHANNEL).emit(message);
      channel.reset();

    } else {

      // Tell other users in room that current connection leaved.
      message = "notif:user-leave " + connid;
      channel.emit(message);

      // Tell lobby that room details have changed
      message = "notif:room-info " + [channel.id, channel.get("count")].join(",");
      domain.getChannel(LOBBY_CHANNEL).emit(message);

    }	

    ....

As you can see signals play a large part in notifying users of changes and also invoking functionality in behaviors, like returning a list of users. This way you can achieve quite a a lot without needing to have your own server setup.

If you want forum to run on your domain you need to upload the behaviors to your hydna domain, see our convenient [tool for this](https://www.hydna.com/documentation/reference/cli/), or use the upload tool on your account page.

###Gravatar###

For avatars we use [gravatar](http://en.gravatar.com/), just have **“http://www.gravatar.com/avatar/youremail?s=40.jpg”**, simple, not very secure but for this demo it’s a nice touch.

###Mobile enabled###

Some final touches with [iScroll](http://cubiq.org/iscroll-4) and [add2home](http://cubiq.org/add-to-home-screen) and we are setup to support IOS devices.

###Benifits###

* All client side, no server functionality needed beyond serving the files.
* Mobile ready, works as it is on an array of mobiles devices as a web app, deploy through [PhoneGap](http://phonegap.com/) to make it into a native app.
* Scale your app as needed, follow usage on your [hydna account page](https://www.hydna.com/account/login/).

We hope this gives a nice introduction to hydna and the power of behaviors, you don't need to use behaviors but as you can see they can be quite powerful. You can of course use our behaviors as a base for you next project, a multiplayer game with room-division perhaps?