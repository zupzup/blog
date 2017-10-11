In the Blockchain world, a light-client is an application which runs its own p2p node and is therefore connected to the whole network without any centralized intermediaries. This is, at least from a decentralized mindset perspective, a desirable property. From a purely technical perspective however, this presents several problems.

Besides the classic performance and reliability issues of a p2p network, one big problem inherent to blockchain systems is, that the data-structure underlying these blockchain systems is usually very large (GBs - TBs), so it's simply not feasible to download it on a mobile device or sometimes even on a desktop. Because of this issue, the concept of light clients was developed. Such a light client, in the case of [ethereum](https://github.com/ethereum/wiki/wiki/Light-client-protocol) only downloads block-headers and verifies a lot less.

With such a light client, it's possible to be a part of the whole p2p network and directly interact with the blockchain by deploying contracts, sending transactions, querying balances etc.

This concept has been working ok for BitCoin in the past and is one of the things which successful blockchain-platforms seem to need at some point. In the ethereum space however, this is, at the time of this blog post, an experimental feature.

When I researched the subject superficially, I didn't find many ethereum projects using a light client yet. Most prominently there is [status](https://status.io/), who wrote their own wrapper of [go-ethereum](https://github.com/ethereum/go-ethereum).

I also found [walleth](https://github.com/walleth/walleth), which is a new project and a Kotlin-based light client using the cross-compiled Android package of go-ethereum.

This cross-compiled package is what I will also use in this blog post, as it seems to be the suggested and most standard way of approaching this problem at the moment. 

I will use react-native in this example, because I'm familiar with react and want to see if wrapping native modules is really [as simple as they say it is](https://facebook.github.io/react-native/docs/native-modules-android.html).

As usual in the ethereum world, there is little to no updated documentation available, adoption of the mobile wrapper seems to be very limited - which means very few examples. Also, testing will be awkward because of the whole blockchain thing, so it's probably going to be a bit of a bumpy ride.

Let's get started!

## Setup

For setting up the react-native application, I usually use [create-react-native-app](https://github.com/react-community/create-react-native-app) and `eject` immediately. This gives us a working, debuggable application which can be started using `npm run android`. For this example, I used a Nexus 5 API 23 emulator.

## Code Example 

The first step is to set up the react-native app to be able to communicate with native Java code. I won't go into many details here, as this process is well documented [here](https://facebook.github.io/react-native/docs/native-modules-android.html).

Initially, we create the following`TestNative.java` file  in `android/app/src/main/java/com/reactnativeethereumwallet`:

```java
package com.reactnativeethereumwallet;

import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactMethod;
import com.facebook.react.bridge.Callback;

public class TestNative extends ReactContextBaseJavaModule {
    public TestNative(ReactApplicationContext reactContext) {
        super(reactContext);
    }

    @Override
    public String getName() {
        return "TestNative";
    }

    @ReactMethod
    public void test(String message, Callback cb) {
        cb.invoke("hello from java: " + message);
    }
}
```

It just registers a method `test` in our `TestNative` module, which takes a message and calls the given callback with a string and the given message to validate that we're able to communicate with Java from react-native.

Then, we need to put this module inside a package - in our case `TestPackage`, right where `TestNative` lives:

```java
package com.reactnativeethereumwallet;

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.JavaScriptModule;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class TestPackage implements ReactPackage {
  @Override
  public List<Class<? extends JavaScriptModule>> createJSModules() {
    return Collections.emptyList();
  }
  @Override
  public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
    return Collections.emptyList();
  }
  @Override
  public List<NativeModule> createNativeModules(
                              ReactApplicationContext reactContext) {
    List<NativeModule> modules = new ArrayList<>();
    modules.add(new TestNative(reactContext));
    return modules;
  }
}
```

This is just some boilerplate for wrapping our module. This package is then registered in the `MainApplication.java` file in the `getPackages` method.

After creating and registering our native module, we can call it from react-native. First, we create a file called `ETH.android.js`

```javascript
import React from 'react';
import { StyleSheet, Text, Alert, Button, View, NativeModules } from 'react-native';

export default class Bla extends React.Component {
  render() {
    return (
      <View style={styles.container}>
        <Button title="button" onPress={() => {
            NativeModules.TestNative.test("Hello!", (str) => {
                Alert.alert(str);
            })
        }}></Button>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    alignItems: 'center',
    justifyContent: 'center',
  },
});
```

This component just renders a button, which, upon pressed, calls `NativeModules.TestNative.test`, the function defined above, with a string and a callback. The callback just alerts the returned string, so we see that passing values between react-native and the android module works.

This `ETH` component needs to be included in `App.js` to be rendered.

So far, so good.

The second step is to get the go-ethereum wrapper to run a light client inside our app. For this part I found it to be easier to use Android Studio or some other Android IDE to be able to debug the native code. 

A bit of a disclaimer here - I'm not an Android developer, in fact I have never created a native Android application, so take all the Android-specific code with a huge grain of salt. ;)

First, we need to add the dependency to the geth-wrapper, the go-ethereum android wrapper to build.gradle:

```bash
compile 'org.ethereum:geth:1.6.7'
```

Now, in `MainActivity.java`, the `Geth` node is started. I tried doing this in the native module, but the application kept crashing (probably because it blocked the UI thread, but I'm not sure). In any case, the node is created and started in the main activity and then, in this case, shared through a global state object called `NodeHolder`:

```java
package com.reactnativeethereumwallet;

import android.os.Bundle;
import android.util.Log;
import com.facebook.react.ReactActivity;
import org.ethereum.geth.Account;
import org.ethereum.geth.Geth;
import org.ethereum.geth.KeyStore;
import org.ethereum.geth.Node;
import org.ethereum.geth.NodeConfig;

public class MainActivity extends ReactActivity {
    @Override
    protected String getMainComponentName() {
        return "reactnativeethereumwallet";
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        try {
            NodeConfig nc = new NodeConfig();
            Node node = Geth.newNode(getFilesDir() + "/.eth1", nc);
            node.start();

            NodeHolder nh = NodeHolder.getInstance();
            nh.setNode(node);

            KeyStore ks = new KeyStore(getFilesDir() + "/keystore", Geth.LightScryptN, Geth.LightScryptP);
            Account newAcc = ks.newAccount("reallyhardpassword");

            android.util.Log.d("keyfile", newAcc.getAddress().getHex());
            nh.setAcc(newAcc);
        } catch (Exception e) {
            Log.d("error: ", e.getMessage());
            e.printStackTrace();
        }
    }
}
```

Alright, this doesn't look so bad. First, we create a `NodeConfig`, which is the configuration object for the light-client. We can configure the network we want to use, the genesis block and many other things. In our case, we just take the default (main network).

Then, the node is created with a file-path. In this path, the node will save all of its files (blockchain-state etc.), the name `/.eth1` is completely arbitrary, you can use whatever you want.

Then the `NodeHolder` singleton is created and the node is set. Afterwards, we also create a new account, which will be saved in the `/keystore` directory. There is some more [documentation](https://github.com/ethereum/go-ethereum/wiki/Mobile:-Account-management) on accounts available.

This account is also shared via the `NodeHolder`.

If you start the application at this point, the node should start and you should see something like the following in the Android Monitor console:

```bash
Starting peer-to-peer node               instance=GethDroid/v1.6.7-stable/android-386/go1.8.3
Allocated cache and file handles         database=/data/user/0/com.reactnativeethereumwallet/files/.eth1/GethDroid/lightchaindata cache=16 handles=16
Initialised chain configuration          ... 
Disk storage enabled for ethash caches   dir=/data/user/0/com.reactnativeethereumwallet/files/.eth1/GethDroid/ethash count=3
Disk storage enabled for ethash DAGs     dir=.ethash                                                                 count=2
Added trusted CHT for mainnet
Loaded most recent local header          number=4086142 hash=a8c492…cffaf2 td=577139722519725357888
Starting P2P networking
Light client mode is an experimental feature
RLPx listener up                         ... 
```

And with a bit of luck and patience, the node should start syncing after a while.

Now, to connect our react-native app to the `Geth` node, we simply use the reference shared in `NodeHolder` within our `TestNative` module:

```java
 try {
    NodeHolder nh = NodeHolder.getInstance();
    Node node = nh.getNode();

    Context ctx = new Context();
    if (node != null) {
        NodeInfo info = node.getNodeInfo();
        EthereumClient ethereumClient = node.getEthereumClient();

        Account newAcc = nh.getAcc();
        BigInt balanceAt = ethereumClient.getBalanceAt(ctx, new Address("0x22B84d5FFeA8b801C0422AFe752377A64Aa738c2"), -1);
        cb.invoke(balanceAt.toString() + " ether found. Account address:" + newAcc.getAddress().getHex());
        return;
    }
    cb.invoke("node was null");
} catch (Exception e) {
    android.util.Log.d("error: ", e.getMessage());
    e.printStackTrace();
}
```

First, we get the node and account from the global state object. Then, we create an `EthereumClient` for the node - this client is then used to interact with the node in a similar way as, for example, web3 would.

With the ethereum client instantiated, we simply query the balance of some address and send it back to react-native together with the address of our created account.
If querying a balance works, it means we are communicating from react-native with our running light-client. That means we can also execute other actions such as interacting with contracts or sending transactions.

And it works!

What I mean by `It works!` is, that if I run the application and the `Geth` node finds peers and starts syncing (which it didn't always manage) and once the syncing is complete (which can take quite a while), then, upon pressing the button I am presented with the correct balance of the hardcoded address, which can be checked on a site like [etherscan](https://etherscan.io/).

That's it. You can find the full code of this simple example [here](https://github.com/zupzup/react-native-ethereum). There are some log statements and commented out lines as well as hardcoded addresses in the code, though, so don't expect a fully cleaned up and ready to use template. ;)

If you start the application, it takes some time (sometimes minutes...) until it connects to other nodes and starts syncing. The syncing itself takes quite a while as well, so this is nothing which you can expect to work out of the box.

## Problems

The biggest problem I see with the cross-compiled go-ethereum light-client wrappers at the moment is adoption. If more people / projects were using it, there would be more documentation and more current examples and projects to get inspiration and help from.

As it stands, however, the API is fully undocumented (limitation of Go-Mobile, which will be fixed at some point) and the software itself seems a bit unstable. There are very few up-to-date examples, tutorials or working Open Source code-bases to learn from.

Testing and Debugging is, as was expected, quite difficult. This probably works better in a native iOS or Android project, but without the source and a documented API there will be some walls you'll run against.

Regarding stability, at the time of creating this post, there was an issue which made syncing with the test network (ropsten) hard/impossible (https://github.com/ethereum/go-ethereum/issues/14851) and when I finally got it to work with the main net, the light client took ~45 minutes to sync to the latest block, but this can probably be tuned - I didn't invest much time researching such optimizations.

## Conclusion

While this was an interesting experience, I have to say there was quite a bit of frustration involved in getting anything to work. The wrapper itself is a good idea I think and the cross-compilation showcases the strength of Go once again, but it needs some more work and, most importantly in my opinion, some projects which use it.

As with all my posts dealing with blockchain and ethereum in particular, I have to say it's just not there yet. The hype and the speculation bubble are there, but the tech is just not anywhere near where it needs to be, in my opinion, for creating stable, real-world systems.

I was, however, quite pleased with how well the react-native part worked. Wrapping the API was the easiest part of this whole exercise, which suggests that, when the underlying light-client is ready for prime-time, it should be trivial to use it from a react-native app.

#### Resources

* [Full Example on Github](https://github.com/zupzup/react-native-ethereum)
* [light client protocol](https://github.com/ethereum/wiki/wiki/Light-client-protocol)
* [react-native android module wrapping](https://facebook.github.io/react-native/docs/native-modules-android.html)
* [go-ethereum](https://github.com/ethereum/go-ethereum)
* [walleth](https://github.com/walleth/walleth)
* [status-go](https://github.com/status-im/status-go)
* [geth mobile docs](https://github.com/ethereum/go-ethereum/wiki/Mobile:-Introduction)
* [block42](http://block42.org/)
