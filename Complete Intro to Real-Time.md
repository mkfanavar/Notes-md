# Complete Intro to Real-Time

## Long Polling

Long polling is really a way of saying "making a lot of requests." There's no special technology here, it's just making an AJAX call on some interval

### Back-End

- The API endpoint should be very fast... pooling end-point should not read-write anything to database... it should use with something very fast like memory-DBs (e.g. Redis)

- You should watch out to not DDOS yourself!

- the implementation don't have anything special. just a simple API endpoint but should be fast. a simple chat endpoint in express:

  ```js
  app.get("/poll", function (req, res){
      res.json({
          msg: getMsgsFromDB(cahtId)
      })
  })
  
  app.post("/poll", function(req, res){
      const {chatInfo, userInfo, body} = req.body;
      writeMsgToDB(chatInfo, userInfo, body);
  })
  ```

### Front-End

- Don't use `setInterval` : when you use `setInterval` you may create a new request when the previous one not finished yet!
- we use `setTimeout` instead. we send next request when the last one finished.

```js
async function getNewMsgs(){
    let json;
    try{
        const res = await fetch("/poll");
        json = await res.json();
    }catch(e){
        // should add backoff strategy here
        console.error("polling error", e);
    }
    allChat = json.msg;
    setTimeout(getNewMsgs, INTERVAL);
}
```



### If we don't need to send request when windows is not focused (minimized or ...)

we should use `requestAnimationFrame` for creating the iteration. with this trick, polling will be paused when user not using the window.

This don't hurt main thread work. `setTimeout` halt any process is already running on main thread and run the callback, but `requestAnimationFrame` respect the thread work.

```js
let timeToMakeNextRequest = 0;
async function rafTimer(time){
    if (timeToMakeNextRequest <= time){
        await getNewMsgs();
        timeToMakeNextRequest = time + INTERVAL;
    }
    
    requestAnimationFrame(rafTimer);
}

requestAnimationFrame(rafTimer);
```

### What should happens when a request got failed?

- retry immediately: the problem with this? this may make your sever down with DDOS.
- backoff: try again with some pause between them. first try immediately if failed try in 3sec => 10sec => 30sec (gmail use this strategy when have problem to connect to server) exponential/linear backoff
