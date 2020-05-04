# Stream iOS (Swift) Encrypted Chat

In this tutorial, we'll build encrypted chat on iOS using Swift. We'll combine Stream Chat and Virgil Security. Both Stream Chat and Virgil make it easy to build a solution with great security with all the features you expect. These two services allow developers to integrate chat that is zero-knowledge. The application embeds Virgil Security's [eThree Kit](https://github.com/VirgilSecurity/virgil-sdk-x) with [Stream Chat's Swift](https://github.com/GetStream/stream-chat-swift) components. All source code for this application is available on [GitHub](https://github.com/psylinse/stream-ios-encrypted-chat).

## What is end-to-end encryption?

End-to-end encryption means that messages sent between two people can only be read by them. To do this, the message is encrypted before it leaves a user's device, and can only be decrypted by the intended recipient.

Virgil Security is a vendor that allows us to create end-to-end encryption via public/private key technology. Virgil provides a platform and a platform that allows us to securely create, store, and provide robust end-to-end encryption.

During this tutorial, we will create a Stream Chat app that uses Virgil's encryption to prevent anyone except the intended parties from reading messages. No one in your company, nor any cloud provider you use, can read these messages. Even if a malicious person gained access to the database containing the messages, all they would see is encrypted text, called ciphertext.

## Building an Encrypted Chat Application

To build this application we'll mostly rely on a few libraries from Stream Chat and Virgil (please check out the dependencies in the source to see what versions). Our final product will encrypt text on the device before sending a message. Decryption and verification will both happen in the receiver's device. Stream's Chat API will only see ciphertext, ensuring our user's data is never seen by anyone else, including you.

To accomplish this, the app performs the following process:

* A user authenticates with your backend.
* The iOS app requests a Stream auth token and API key from the `backend`. The Swift app creates a [Stream Chat Client](https://getstream.io/chat/docs/#init_and_users) for that user.
* The mobile app requests a Virgil auth token from the `backend` and registers with Virgil. This generates their private and public key. The private key is stored locally, and the public key is stored in Virgil.
* Once the user decides who they want to chat with the app creates and joins a [Stream Chat Channel](https://getstream.io/chat/docs/#initialize_channel).
* The app asks Virgil's API, via the eThree kit, for the receiver's public key.
* The user types a message and sends it to stream. Before sending, the app passes the receiver's public key to Virgil to encrypt the message. The message is relayed through Stream Chat to the receiver. Stream receives ciphertext, meaning they can never see the original message.
* The receiving user decrypts the sent message using Virgil. When the message is received, app decrypts the message using the Virgil and this is passed along to Stream's React components. Virgil verifies the message is authentic by using the sender's public key.

While this looks complicated, Stream and Virgil do most of the work for us. We'll use Stream's out of the box Swift UI components to render the chat UI and Virgil to do all of the cryptography and key management. We simply combine these services. 

The code is split between the iOS frontend contained in the `ios` folder and the Express (Node.js) backend is found in the `backend` folder. See the `README.md` in each folder to see installing and running instructions. If you'd like to follow along with running code, make sure you get both the `backend` and `flutter` running before continuing.

Let's walk through and look at the important code needed for each step.

## Prerequisites

Basic knowledge of iOS (Swift) and Node.js is required to follow this tutorial. This code is intended to run locally on your machine. 

You will need an account with [Stream](https://getstream.io/accounts/signup/) and [Virgil](https://dashboard.virgilsecurity.com/signup). Once you've created your accounts, you can place your credentials in `backend/.env` if you'd like to run the code. You can use `backend/.env.example` as a reference for what credentials are required. You also need to set up the api root url in `Account.swift` using ngrok or something similar. Please see the READMEs in each directory for more information.

## Step 0. Setup the Backend

For our Swift app to securely interact with Stream and Virgil, the `backend` provides three endpoints:

* `POST /v1/authenticate`: This endpoint generates an auth token that allows the React frontend to communicate with the other endpoints. To keep things simple, this endpoint allows the client to be any user. The frontend tells the backend who it wants to authenticate as. In your application, this should be replaced with real authentication appropriate for your app.

* `POST /v1/stream-credentials`: This returns the data required for the iOS app to establish a session with Stream. In order return this info we need to tell Stream this user exists and ask them to create a valid auth token:
  
  ```javascript
  // backend/src/controllers/v1/stream-credentials.js
  exports.streamCredentials = async (req, res) => {
    const data = req.body;
    const apiKey = process.env.STREAM_API_KEY;
    const apiSecret = process.env.STREAM_API_SECRET;
  
    const client = new StreamChat(apiKey, apiSecret);
  
    const user = Object.assign({}, data, {
      id: `${req.user.sender}`,
      role: 'admin',
      image: `https://robohash.org/${req.user.sender}`,
    });
    const token = client.createToken(user.id);
    await client.updateUsers([user]);
    res.status(200).json({ user, token, apiKey });
  }
  ```
  
  The response payload has this shape:
  
  ```json
  {
    "apiKey": "<string>",
    "token": "<string>",
    "user": {
      "id": "<string>",
      "role": "<string>",
      "image": "<string>"
    }
  } 
  ```
  
   * `apiKey` is the stream account identifier for your Stream instance. Needed to identify what account your frontend is trying to connect with.
   * `token` JWT token to authorize the frontend with Stream.
   * `user`: This object contains the data that the frontend needs to connect and render the user's view.

* `POST /v1/virgil-credentials`: This returns the authentication token used to connect the frontend to Virgil. We use the Virgil Crypto SDK to generate a valid auth token for us:
  
  ```javascript
  // backend/src/controllers/v1/virgil-credentials.js
  async function getJwtGenerator() {
    await initCrypto();
  
    const virgilCrypto = new VirgilCrypto();
    // initialize JWT generator with your App ID and App Key ID you got in
    // Virgil Dashboard.
    return new JwtGenerator({
      appId: process.env.VIRGIL_APP_ID,
      apiKeyId: process.env.VIRGIL_KEY_ID,
      apiKey: virgilCrypto.importPrivateKey(process.env.VIRGIL_PRIVATE_KEY),
      accessTokenSigner: new VirgilAccessTokenSigner(virgilCrypto)
    });
  }
  
  const generatorPromise = getJwtGenerator();
  
  exports.virgilCredentials = async (req, res) => {
    const generator = await generatorPromise;
    const virgilJwtToken = generator.generateToken(req.user.sender);
  
    res.json({ token: virgilJwtToken.toString() });
  };
  ```
  
  In this case, the frontend only needs the auth token.

* `GET /v1/users`: Endpoint for returning all users. This exists just to get a list of people to chat with. Please refer to the source if you're curious how this is built. Please note the `backend` uses in-memory storage, so if you restart it will forget all of the users.

## Step 1. User Authenticates With Backend

The first step is to authenticate a user and get our Stream and Virgil credentials. To keep thing simple, we'll have a simple form that allows you to log in as anyone:

![](images/login.png)

This is a simple form that takes any arbitrary name, effectively allowing us to log in as anyone (obviously, this should be an appropriate authentication method for you application). First let's add to `Main.storyboard`. We add a "Login View Controller" scene that's backed by a custom controller `LoginViewController` (to be defined). This should be nested beneath a Navigation Controller. Your storyboard should look something like this:

![](images/login-storyboard.png)

The form is a simple `Stack View` with a `username` field and submit button. Let's look at our custom `LoginViewController`:

```swift
// ios/EncryptedChat/LoginViewController.swift:3
class LoginViewController: UIViewController {    
    @IBOutlet weak var usernameField: UITextField!
    
    @IBAction func login(_ sender: Any) {
        guard let userId = usernameField.text, !userId.isBlank else {
            usernameField.placeholder = " ⚠️ User id"
            return
        }
        
        Account.shared.login(userId) {
            DispatchQueue.main.async {
                self.performSegue(withIdentifier: "UsersSegue", sender: self)
            }
        }
    }
}
```

The `usernameField` is bound to the Storyboard's `Username Field` and the login method is bound to `Login` button. When a user clicks login we check if there's a username and if so, we login via `Account.shared.login`. `Account` is a shared object that will store our credentials for future backend interactions. Once the login is performed we initiate the `UsersSegue` which boots our next scene. We'll see how this is done in a second, but first let's see how we login.

Here's how we define `Account`:

```swift
// ios/EncryptedChat/Account.swift:5
class Account {
    public static let shared = Account()

    let apiRoot = "https://623a2139.ngrok.io" // make sure the backend is running and accessible via ngrok or something similar
    var authToken: String? = nil
    var userId: String? = nil

    public func login(_ userId: String, completion: @escaping () -> Void) {
        AF
            .request("\(apiRoot)/v1/authenticate",
                     method: .post,
                     parameters: ["user" : userId],
                     encoder: JSONParameterEncoder.default)
            .responseJSON { response in
                let body = response.value as! NSDictionary
                let authToken = body["authToken"]! as! String

                self.authToken = authToken
                self.userId = userId
                
                self.setupStream(completion)
        }
    }

    //...
}
```

First, we set up our shared object that will store our login state in the `authToken` and `userId` properties. Note, `apiRoot` which is how our app connects to our backend running on `localhost`. Please follow the instructions in the `backend` to run it. Use something like `ngrok` to facilitate the connection to `localhost`. Our `login` method uses Alamofire (`AF`) to make a `post` request to our backend with the user to log in. Upon a success, we store the `authToken` and `userId` and call `setupStream`. 

The method `setupStream` initializes our Stream Chat client. Let's see it's implementation:

```swift
// ios/EncryptedStream/Account.swift:39
private func setupStream(_ completion: @escaping () -> Void)  {
    AF
        .request("\(apiRoot)/v1/stream-credentials",
                  method: .post,
                  headers: ["Authorization" : "Bearer \(authToken!)"])
        .responseJSON { response in
            let body = response.value as! NSDictionary
            let token = body["token"]! as! String
            let apiKey = body["apiKey"]! as! String
            
            Client.config = .init(apiKey: apiKey, logOptions: .info)
            Client.shared.set(
                user: User(id: self.userId!),
                token: token
            )
            
            self.setupVirgil(completion)
    }
}
```

We call to the `backend` with our credentials from `login`. We get back a `token`, which is a Stream Chat frontend token. This token allows our mobile application to communicate directly with Stream without going through our `backend`. We also get a `apiKey` which identifies the Stream account we're using. We this data to initialize our Stream `Client` instance and set the user. Last, we initalize virgil via `setupVirgil`:

```swift
// ios/EncryptedChat/Account.swift:59
private func setupVirgil(_ completion: @escaping () -> Void) {
    AF
        .request("\(apiRoot)/v1/virgil-credentials",
                  method: .post,
                  headers: ["Authorization" : "Bearer \(authToken!)"])
        .responseJSON { response in
            let body = response.value as! NSDictionary
            let token = body["token"]! as! String
      
            VirgilClient.configure(identity: self.userId!, token: token)
            
            completion()
    }
}
```

TODO