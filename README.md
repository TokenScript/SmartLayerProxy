# SmartLayerProxy
## A universal, authenticated proxy for safely connecting TokenScript to remote services

### Why is this needed? Don't we already have things like this?
Similar services exist but lack the ability for permissionless setup, have at best static token authentication and integration is non-trivial. The default go-to is the excellent [Blynk](https://github.com/blynkkk). This is easy to use but as a proprietry solution it's not compatible with the premissionless style of operation required for our library.
As to, "why do we need this service to connect to (for exmaple) IoT devices" - that is due to traversal of the NAT firewall an IoT device is inevitably behind. The IoT device must contact an external server in order to receive instructions.

### How will it work?
Ultimately the SmartLayer Proxy will run on the SmartLayer over a distributed service where participants are rewarded for latency and availability.

### How do I connect an IoT device to a TokenScript?
The IoT device requires an ethereum private key. This can be issued by an external source (eg Web3j, ethers.js or even MetaMask) or internally by Web3E. Once issued we know the ethereum address corresponding to that private key. This address is the effective address of the device itself on the smart layer network (think of it like the device's URL).

### Draft Implementation
This repo holds the draft implementation for SmartLayer comms and possibly serve for legacy devices.

The device will initiate contact with the proxy server using a login packet. The server will issue a 16 byte SecureRandom challenge. The device signs that challenge and replies with the signature. The server now assigns the EC recovered address to the device address.

Devices are communicated with via TokenScript by specifying the device address in the addressing call. Consider a simple IoT controlled door that can be unlocked using an NFT - this consists of two processes:

1: A means to verify that the potential user controls the private key of the wallet to which the NFT 'Entry Key' has been sent.
2: A means for the IoT device to operate the associated door.

For process 1, the simplest method is to request the user sign a message with their key and then query the balance of the recovered wallet for the 'Entry Key' NFT. Therefore we require a challenge/response type of communications.

The flow would look like this:

1. User contacts IoT device for the message to be signed (we will call this the challenge).
2. User uses their wallet to sign this challenge and then return the signature to the IoT device.
3. IoT device now operates the unlock mechanism.

In this exchange there is no trivial remote attack path other than to copy and attempt to replay the signature (which can be solved by using a new challenge after every signature is received whether it passes or not). A logjam style attack will be irrelevant as the ECDSA signature is required to trigger the device to unlock. The path to the server is protected using https certificate and the JavaScript engine itself that is running the TokenScript file.


The TokenScript calls the device using an API:

``` JavaScript
var iotAddr = "0x0071234567890abcdef1234567890abcdef01234" // the address of your IoT device, from
                                                           // 'How do I connect an IoT device to a TokenScript?'
var serverAddr = "https://scriptproxy.smarttokenlabs.com"; // Fixed server endpoint
...
fetch(`${serverAddr}:8433/api/${iotAddr}/getChallenge`)
...
```

To fetch the challenge. Once the wallet has signed this challenge the signature is returned to the device:

``` JavaScript
...
fetch(`${serverAddr}:8433/api/${iotAddr}/checkSignature?sig=${value}`)
...
```

The IoT device now can perform ec-recover on the signature and check they hold the NFT which the device has been programmed to check for.

For the [Espressif ESP32](https://www.espressif.com/en/products/socs/esp32) family of devices we have a Web3 library [Web3E](https://github.com/AlphaWallet/Web3E) which gives you the ability to call functions, send transactions and sign messages and in addition contains an easy to setup integration for the SmartLayer connection which handles login, authentication and API call shim.

To implement on the ESP32 side (Assuming you are using PlatformIO):

Include Web3E in your dependencies:

```
lib_deps =
  # Using a library name
  Web3E
```

Declare Globals:

``` C++
TcpBridge *tcpConnection;
Web3 *web3;
KeyID *keyID;
```

Set up Web3E in your setup (eg):

``` C++
Web3 *web3 = new Web3(SEPOLIA_ID);
keyID = new KeyID(web3, DEVICE_PRIVATE_KEY); // Private key for this device if you are fixing it (easiest way).
tcpConnection = new TcpBridge();
tcpConnection->setKey(keyID, web3);
tcpConnection->startConnection();
```

Now set up the API callback so you can act on received instructions from the any Token service (in this case a call from a TokenScript):

``` C++
enum APIRoutes
{
  api_unknown,
  api_getChallenge,
  api_checkSignature,
  api_checkSignatureLock,
  api_checkMarqueeSig,
  api_end
};

std::map<std::string, APIRoutes> s_apiRoutes;

void Initialize()
{
  s_apiRoutes["getChallenge"] = api_getChallenge;
  s_apiRoutes["checkSignature"] = api_checkSignature;
  s_apiRoutes["checkSignatureLock"] = api_checkSignatureLock;
  s_apiRoutes["end"] = api_end;
}

// Callback to handle routes as they are called
std::string handleTCPAPI(APIReturn* apiReturn)
{
  switch (s_apiRoutes[apiReturn->apiName])
  {
  case api_getChallenge:
    Serial.println(currentChallenge.c_str());
    udpBridge->sendResponse(currentChallenge, methodId);
    break;
  case api_checkSignature:
  {
    //EC-Recover address from signature and challenge
    string address = Crypto::ECRecoverFromPersonalMessage(&apiReturn->params["sig"], &currentChallenge);  
		//Check if this address has our entry token
    boolean hasToken = QueryBalance(&address);
    updateChallenge(); //generate a new challenge after each check
    if (hasToken)
    {
      udpBridge->sendResponse("pass", methodId);
      OpenDoor(); //Call your code that opens a door or performs the required 'pass' action
    }
    else
    {
      udpBridge->sendResponse("fail: doesn't have token", methodId);
    }
  }
  break;
}
```

And ensure the Bridge is called from your loop or from a thread:

``` C++
void loop()
{
...
tcpConnection->checkClientAPI(&handleTCPAPI); //Pass in callback function so the TCP Bridge can call your API handler
...
}
```

There is a video on YouTube where I explain how the SmartLayer works with diagrams here: https://www.youtube.com/watch?v=WjuDTe8eik4

This video is out of date now as the system is much improved since then.

We have also used this technology to create a NFT chat board, where people with the same NFT collection can chat about what's happening. The boards are public but the users are required to prove they own an NFT in the collection in order to post messages.

