# райтуп

We have an apk file. From task description we know that it's a quiz app and all we need is  to answer 1000 questions to get the flag.
Let's check traffic using the BurpSuit's proxy. Here we have sniffed websocket queries:
[pic]
Logic is quite simple. Server sends encoded question and options, the app sends answer (0-3). But the strange field "correctAnswer" seems to be encoded with the same method as other fields.
Let's decompile apk and look through java code.
```sh
$ d2j-dex2jar client.apk
$ jd-gui client-dex2jar.jar
```
As we can see, data is encoded with AES and there are two parameters, that some algorithm generates. But зачем думать и реверсить его, если можно использовать фриду. 

Downloading frida:
```sh
$ wget https://github.com/frida/frida/releases/download/12.8.20/frida-gadget-12.8.20-android-{arch}.so.xz  # <-- arch of your phone e.g. arm64, arm, ...
$ unxz frida-gadget-12.8.20*
$ pip install frida-tools
```
Decompiling apk and placing frida gadget
```sh
$ apktool d client.apk
$ mkdir -p client/lib/{you-arch} 
$ mv frida-gadget-12.8.20 client/lib/{you-arch}/libfrida-gadget.so
```
 Now inject frida-gadget to main activity in *client/smali/wtf/riceteacatpanda/quiz/MainActivity.smali* after 16th line
```smali
    const-string v0, "frida-gadget" # YES, name is like .so file but without lib
    invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V
```
Build & sign modified apk
```sh
$ apktool b -o repackaged.apk client
$ keytool -genkey -v -keystore custom.keystore -alias mykeyaliasname -keyalg RSA -keysize 2048 -validity 10000
$ jarsigner -sigalg SHA1withRSA -digestalg SHA1 -keystore custom.keystore -storepass {password here} repackaged.apk mykeyaliasname
```

Now let's write the script for Frida. Frida lets us override java methods
```javascript 
Java.perform(function x() {

    var correctEncoded = '';
    var decoded = -1;
    var sent=false;


    // Оверрайдим конструктур JSONObject. Можно видеть в декомпиленном коде, что он создается на каждый новый вопрос.
    Java.use('org.json.JSONObject').$init.overload('java.lang.String').implementation = function(str) {
        var obj = JSON.parse(str);
        if (obj.correctAnswer != undefined){
            correctEncoded = obj.correctAnswer;
            sent=false;
        }
        return this.$init(str);
    };

    // нам удобно подцепиться к методу doFinal, так как он юзается в момент, когда ключи уже сгенерированы и инстанс Cipher готов к декодированию
    Java.use("javax.crypto.Cipher").doFinal.overload('[B').implementation = function(bytes){
        decoded = parseInt( "" + (Java.use('java.lang.String').$new(this.doFinal(Java.use('android.util.Base64').decode(correctEncoded, 0))))); // decoding correctAnswer

        if (!sent){ // чтобы не кидало несколько ответов на один вопрос
            var kr = Java.use('nw').a();  // это просто скопировали из декомпилера, производит отправку и переход к новому вопросу
            var data = "{\"method\":\"answer\",\"answer\":" + decoded + "}";
            kr.a(data);
            sent = true;
        }
        return this.doFinal(bytes);
    };
});
```

Run app, run *adb*, click login and
```sh
$ frida -U gadget -l script.js
```
done!
