# Przelewy24 library documentation - Android

For general information on the operation of Przelewy24 mobile libraries, visit:

- [https://github.com/przelewy24/p24-mobile-lib-doc](https://github.com/przelewy24/p24-mobile-lib-doc)

## 1. Project configuration

The first step is to set the value `minSdkVersion=14` in file `build.gradle`

### Adding dependencies

In the Android Studio environment, it is possible to add a library module by using the command: „File → New → New module...”. From the list „New module” select „Import .JAR or .AAR Package” and click „Next”. In the field „File name” provide access path to file `p24Lib.aar`. As a „Subproject name”, provide „p24Lib” and click „Finish”.

he next step is to add a dependency to the created library module by modifying the file `build.gradle` and placing the following entry in the section „dependencies”:

`compile project(':p24lib')`

Since the library uses the AppCompat v7 library, the following dependency must be added:

`compile 'com.android.support:appcompat-v7:26.+'`

Below is an example of a „dependencies” section:

```gradle

dependencies {
	//other dependencies
    compile 'com.android.support:appcompat-v7:26.+'
    compile project(':p24Lib')
}

```

### Definition of AndroidManifest file

Add the following to `AndroidManifest.xml` file:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

In case the SMS code paste function is used, add also:

```xml
<uses-permission android:name="android.permission.RECEIVE_SMS"/>
```

Next, in `application` section, add `TransferActivity`:

```xml
<activity android:name=„pl.przelewy24.p24lib.transfer.TransferActivity"
          android:configChanges="orientation|keyboard|keyboardHidden"
          android:theme="@style/Theme.AppCompat.Light.DarkActionBar”/>
```

and `PaymentSettingsActivity`:

```xml
<activity android:name="pl.przelewy24.p24lib.settings.PaymentSettingsActivity"
          android:configChanges="orientation|keyboard|keyboardHidden"
          android:theme="@style/Theme.AppCompat.Light.DarkActionBar”/>
```

__All Activities in the library draw on the AppCompatActivity, which is why  „Theme.AppCompat.*” group styles as well as derivative styles should be used__

n the case of default Activity settings, the WebView will get reloaded during the revolution of the library display screen, which may cause a return from the bank’s website to the list of payment forms and render further transaction processing impossible. In order to prevent the reloading of the library window, the following parameter must be set:

```xml
android:configChanges="orientation|keyboard|keyboardHidden"
```
Below is an example of an `AndroidManifext.xml` file:

```xml

<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="pl.przelewy24.p24example"
    android:versionCode="1"
    android:versionName="1.0.0">

	<!--other permissions-->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <!--optional-->
    <uses-permission android:name="android.permission.RECEIVE_SMS"/>

    <application >

		<!--other activities-->

        <activity android:name="pl.przelewy24.p24lib.transfer.TransferActivity"
                  android:configChanges="keyboardHidden|orientation|keyboard|screenSize"
                  android:theme="@style/Theme.AppCompat.Light.DarkActionBar"/>

        <activity android:name="pl.przelewy24.p24lib.settings.PaymentSettingsActivity"
                  android:configChanges="keyboardHidden|orientation|keyboard|screenSize"
                  android:theme="@style/Theme.AppCompat.Light.DarkActionBar"/>

    </application>

</manifest>


```

## 2. trnDirect transaction call

In order to call the transaction, the following parameters must be set using the builder class and providing the Merchant ID and the CRC key:

```java
TransactionParams transactionParams = new TransactionParams.Builder()
           .merchantId(XXXXX)
           .crc(XXXXXXXXXXXXX)
           .sessionId(XXXXXXXXXXXXX)
           .amount(1)
           .currency("PLN")
           .description("test payment description")
           .email("test@test.pl")
           .country("PL")
           .client("John Smith")
           .address("Test street")
           .zip("60-600")
           .city("Poznan")
           .phone("1246423234")
           .language("pl")
           .build();
```

Optional parameters:

```java
builder.urlStatus("https://XXXXXX")
       .method(25)
       .timeLimit(90)
       .channel(1)
       .transferLabel("transfer label")
       .shipping(0);
```
Next, an object with the transaction call parameters should be created that will be applicable to the specific method:

```java
TrnDirectParams params = TrnDirectParams.create(transactionParams);
```

Optionally, the transaction call may be set at the Sandbox server:

```java
params.setSandbox(true);
```

Yet another option is to add saved library settings for bank websites (mobile styles at the banks’ websites – turned on by default, Should the library remember logins and passwords?, Should the library automatically paste sms passwords to the transaction confirmation form at the bank):

```java
SettingsParams settingsParams = new SettingsParams();
settingsParams.setEnableBanksRwd(true);
settingsParams.setSaveBankCredentials(true);
settingsParams.setReadSmsPasswords(true);
params.setSettingsParams(settingsParams);
```

In case  `setReadSmsPasswords` is set as `true`, the following should be added to the manifest:

```xml
<uses-permission android:name="android.permission.RECEIVE_SMS"/>
```

With the configurational objects complete, one may proceed to call Activity for a transaction. The initiation looks as follows:

```java
Intent intent = TransferActivity.getIntentForTrnDirect(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

In order to serve the transaction result, one must modify `Activity.onActivityResult`:

```java
@Override
protected void onActivityResult(int reqCode, int resCode, Intent data) {
    super.onActivityResult(reqCode, resCode, data);
    if (reqCode == TRANSACTION_REQUEST_CODE) {
        if (resCode == RESULT_OK) {
            TransferResult result = TransferActivity.parseResult(data);

            switch (result) {
                case SUCCESS:
                    // success
                    break;
                case ERROR:
                    //error
                    break;
            }

        } else {
            //cancel
        }
    }
}
```
`TransferActivity` yields only information regarding the completion of the transaction. It need not mean that the transaction has been verified by the partner’s server. That is why, each time the `SUCCESS` status is obtained, the application should inquire its own backend about the transaction status.


## 3. trnRequest transaction call

During the registration with the "trnRegister" method, parameter p24_mobile_lib=1 should be provided, which allows Przelewy24 to classify the transaction as a mobile transaction. A Token registered without this parameter will not work in the mobile application (an error will appear upon return to the bank and the library file will not detect payment completion).

**NOTE!**

 > When registering a transaction which is to be carried out in a mobile library, remember about the additional parameters:
- `p24_channel` – unless set, the library will feature the payment options „traditional transfer” and „use prepayment”, which are unnecessary in case of mobile payments. In order to deactivate them, use flags that disregard these forms (e.g. value 3 – payments and cards, default entry setting, directly with parameters)
- `p24_method` – if a given transaction in the library is to have a specific, preset method of payment, this method must be selected during the registration
- `p24_url_status` - the address to be used for transaction verification by the partner’s server once the payment process in the mobile library is finished

The transaction parameters must be set using the token of a transaction registered earlier. Alternatively, the sandbox server and bank configuration may be set:

```java
TrnRequestParams params = TrnRequestParams
                      .create(„XXXXXXXXXX-XXXXXX-XXXXXX-XXXXXXXXXX”)
                      .setSandbox(true)
                      .setSettingsParams(settingsParams);
```

Next, `Intent` should be created to call transaction `Activity` and run it:

```java
Intent intent = TransferActivity.getIntentForTrnRequest(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

The transaction result should be served in the same way as in the case of "trnDirect".

## 4. Express transaction call

The transaction parameters must be set using the url obtained during the registration of the transaction with Express. The transaction must be registered as mobile.

```java
ExpressParams params = ExpressParams.create(expresTransactionUrl);
```

Next, create Intent to call Activity and run it:

```java
Intent intent = TransferActivity.getIntentForExpress(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

The transaction result should be served in the same way as in the case of "trnDirect".

## 5. Passage 2.0 transaction call

The transaction parameters must be set in the same way as for "trnDirect". A properly prepared cart object should be added:

```java
PassageCart passageCart = PassageCart.create();
PassageItem.Builder builder = new PassageItem.Builder()
           .name("Product name 1")
           .description("Product description 1")
           .number(1)
           .price(10)
           .quantity(2)
           .targetAmount(20)
           .targetPosId(XXXXXX);

passageCart.addItem(builder.build());
```

```java
TransactionParams transactionParams = new TransactionParams.Builder()
            ...
           .passageCart(passageCart)
           .build();
```

The transaction call and result parsing proceed in the same way as in the case of "trnDirect".
