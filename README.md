# Sample Vonage Video NodeJS Server App

<img src="https://assets.tokbox.com/img/vonage/Vonage_VideoAPI_black.svg" height="48px" alt="Tokbox is now known as Vonage" />

This simple server app shows you how to use [Vonage Video Node Server SDK](https://developer.vonage.com/video/server-sdks/node) to create Vonage sessions, generate tokens for those sessions, archive (or record) sessions, and download those archives.

## Quick deploy

### Heroku

Heroku is a PaaS (Platform as a Service) that can be used to deploy simple and small applications for free. To easily deploy this repository to Heroku, sign up for a Heroku account and click this button:

<a href="https://heroku.com/deploy?template=https://github.com/Vonage-Community/sample-video-node-learning_server/" target="_blank">
<img src="https://www.herokucdn.com/deploy/button.png" alt="Deploy">
</a>

Heroku will prompt you to add your Vonage Application ID and Vonage Private Key, which you can
obtain at the [Vonage Dashboard](https://dashboard.nexmo.com).

### Railway

[Railway](https://railway.app/) is a deployment platform where you can provision infrastructure, develop with that infrastructure locally, and then deploy to the cloud.

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/template/GHmSB0)

Railway will prompt you to add your Vonage Application ID and Vonage Private Key, which you can
obtain at the [Vonage Dashboard](https://dashboard.nexmo.com).

## Requirements

- [Node.js](https://nodejs.org/)

## Installing & Running on localhost

  1. Clone the app by running the command

		  git clone git@github.com:Vonage-Community/sample-video-node-learning_server.git

  2. `cd` to the root directory.
  3. Run `npm install` command to fetch and install all npm dependencies.
  4. Next, rename the `.envcopy` file located at the root directory to `.env`, and enter in your Vonage App ID and Private Key as indicated:

      ```
        # enter your Vonage Application ID after the '=' sign below
        API_APPLICATION_ID=your_application_id
        
        # enter your Vonage Private Key as a string or path to the file after the '=' sign below
        PRIVATE_KEY=your_private_key
        
        # OR enter your Vonage Private Key as a base64-encoded value ( Linux: cat private.key | base64 -w 0 Mac: cat private.key | base64 -b 0 ) after the '=' sign below
        PRIVATE_KEY64=your_private_key64
      ```
    
  4. Run `npm start` to start the app.
  5. Visit the URL <http://localhost:3000/session> in your browser. You should see a JSON response containing the Vonage Application ID, session Id, and token.

## Exploring the code 

The `routes/index.js` file is the Express routing for the web service. The rest of this tutorial
discusses code in this file.

In order to navigate clients to a designated meeting spot, we associate the [Session ID](https://developer.vonage.com/video/overview#sessions) to a room name which is easier for people to recognize and pass. For simplicity, we use a local associated array to implement the association where the room name is the key and the [Session ID](https://developer.vonage.com/video/overview#sessions) is the value. For production applications, you may want to configure a persistence (such as a database) to achieve this functionality.

### Generate/Retrieve a Session ID

The `GET /room/:name` route associates an OpenTok session with a "room" name. This route handles the passed room name and performs a check to determine whether the app should generate a new session ID or retrieve a session ID from the local in-memory hash. Then, it generates a Vonage Video token for that session ID. Once the Application ID, session ID, and token are ready, it sends a response with the body set to a JSON object containing the information.

```javascript
if (roomToSessionIdDictionary[roomName]) {
    sessionId = roomToSessionIdDictionary[roomName];

    // generate token
    token = vonage.video.generateClientToken(sessionId);
    res.setHeader('Content-Type', 'application/json');
    res.send({
        applicationId: appId,
        sessionId: sessionId,
        token: token
    });
}
// if this is the first time the room is being accessed, create a new session ID
else {
    try {
        const session = await vonage.video.createSession({ mediaMode:"routed" });

        // now that the room name has a session associated wit it, store it in memory
        // IMPORTANT: Because this is stored in memory, restarting your server will reset these values
        // if you want to store a room-to-session association in your production application
        // you should use a more persistent storage for them
        roomToSessionIdDictionary[roomName] = session.sessionId;

        // generate token
        token = vonage.video.generateClientToken(session.sessionId);
        res.setHeader('Content-Type', 'application/json');
        res.send({
            applicationId: appId,
            sessionId: session.sessionId,
            token: token
        });
    } catch(error) {
        console.error("Error creating session: ", error);
        res.status(500).send({ error: 'createSession error:' + error });
    }
}
```

The `GET /session` routes generates a convenient session for fast establishment of communication.

```javascript
router.get('/session', function(req, res, next) { 
  res.redirect('/room/session'); 
}); 
```

### Start an [Archive](https://developer.vonage.com/video/guides/archiving/overview)

A `POST` request to the `/archive/start` route starts an archive recording of a Vonage Video session.
The session ID is passed in as JSON data in the body of the request

```javascript
router.post('/archive/start', async function (req, res) {
    console.log('attempting to start archive');
    const json = req.body;
    const sessionId = json.sessionId;
    try {
        const archive = await vonage.video.startArchive(sessionId, { name: findRoomFromSessionId(sessionId) });
        console.log("archive: ", archive);
        res.setHeader('Content-Type', 'application/json');
        res.send(archive);
    } catch (error){
        console.error("error starting archive: ",error);
        res.status(500).send({ error: 'startArchive error:' + error });
    }
});
```

> Note: You can only create an archive for sessions that have at least one client connected. Otherwise,
the app will respond with an error.

### Stop an Archive

A `POST` request to the `/archive/:archiveId/stop` route stops an archive's recording.
The archive ID is returned by the call to the `archive/start` endpoint.

```javascript
router.post('/archive/:archiveId/stop', async function (req, res) {
    const archiveId = req.params.archiveId;
    console.log('attempting to stop archive: ' + archiveId);
    try {
        const archive = await vonage.video.stopArchive(archiveId);
        res.setHeader('Content-Type', 'application/json');
        res.send(archive);
    } catch (error){
        console.error("error stopping archive: ",error);
        res.status(500).send({ error: 'stopArchive error:', error });
    }
});
```

### View an Archive

A `GET` request to `'/archive/:archiveId/view'` redirects the requested clients to
a URL where the archive gets played.

```javascript
router.get('/archive/:archiveId/view', async function (req, res) {
    const archiveId = req.params.archiveId;
    console.log('attempting to view archive: ' + archiveId);
    try {
        const archive = await vonage.video.getArchive(archiveId);
        if (archive.status === 'available') {
            res.redirect(archive.url);
        } else {
            res.render('view', { title: 'Archiving Pending' });
        }
    } catch (error){
        console.log("error viewing archive: ",error);
        res.status(500).send({ error: 'viewArchive error:' + error });
    }
});
``` 

### Get Archive information

A `GET` request to `/archive/:archiveId` returns a JSON object that contains all archive properties, including `status`, `url`, `duration`, etc. For more information, see [here](https://developer.vonage.com/video/guides/archiving/overview).

```javascript
router.get('/archive/:archiveId', async function (req, res) {
    const archiveId = req.params.archiveId;
    // fetch archive
    console.log('attempting to fetch archive: ' + archiveId);
    try {
        const archive = await vonage.video.getArchive(archiveId);
        // extract as a JSON object
        res.setHeader('Content-Type', 'application/json');
        res.send(archive);
    } catch (error){
        console.error("error getting archive: ",error);
        res.status(500).send({ error: 'getArchive error:' + error });
    }
});
```

### Fetch multiple Archives

A `GET` request to `/archive` with optional `sessionId`, `count` and `offset` params returns a list of JSON archive objects. For more information, please check [here](https://developer.vonage.com/video/guides/archiving/overview).

Examples:
```javascript
GET /archive // fetch up to 1000 archive objects
GET /archive?sessionId=1_MX42...xQVJmfn4 // fetch archive(s) with session ID
GET /archive?count=10  // fetch the first 10 archive objects
GET /archive?offset=10  // fetch archives but first 10 archive objetcs
GET /archive?count=10&offset=10 // fetch 10 archive objects starting from 11th
```

## More information

This sample app does not provide client-side functionality
(for connecting to Vonage Video sessions and for publishing and subscribing to streams).
It is intended to be used with the Vonage Video tutorials for Web, iOS, iOS-Swift, or Android:

* [Web](https://developer.vonage.com/video/client-sdks/web)
* iOS coming soon
* Android coming soon

## Development and Contributing

Interested in contributing? We :heart: pull requests! See the [Contribution](CONTRIBUTING.md) guidelines.

## Getting Help

We love to hear from you so if you have questions, comments or find a bug in the project, let us know! You can either:

- Open an issue on this repository
- See <https://developer.vonage.com/community/> for support options
- Tweet at us! We're [@VonageDev](https://twitter.com/VonageDev) on Twitter
- Or [join the Vonage Developer Community Slack](https://developer.vonage.com/community/slack)

## Further Reading

- Check out the Developer Documentation at <https://developer.vonage.com/>
