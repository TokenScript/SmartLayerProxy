# SmartLayerProxy
## A universal, authenticated proxy for safely connecting TokenScript to remote services

### Why is this needed? Don't we already have things like this?
Similar services exist but lack the ability for permissionless setup, have at best static token authentication and integration is non-trivial.
As to, "why do we need this service to connect to (for exmaple) IoT devices" - that is due to traversal of the NAT firewall an IoT device is inevitably behind. The IoT device must contact an external server in order to receive instructions.

### How do I connect an IoT device to a TokenScript?
The IoT device requires an ethereum private key. This can be issued by an external source (eg Web3j, ethers.js or even MetaMask) or internally by Web3E. Once issued we know the ethereum address corresponding to that private key. This address is the effective address of the device itself on the smart layer network (think of it like the device's URL).

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

For the [Espressif ESP32](https://www.espressif.com/en/products/socs/esp32) family of devices we have a Web3 library [Web3E](https://github.com/AlphaWallet/Web3E) which contains an easy to setup integration for the SmartLayer connection which handles login, authentication and API call shim.
