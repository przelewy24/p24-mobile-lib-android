# Przelewy24 library documentation - Android
![](https://raw.githubusercontent.com/przelewy24/p24-mobile-lib-android/master/libVerImg.svg?sanitize=true)

For general information on the operation of Przelewy24 mobile libraries, visit:

- [https://github.com/przelewy24/p24-mobile-lib-doc](https://github.com/przelewy24/p24-mobile-lib-doc)

To see implementation example please check the example project:

- [https://github.com/przelewy24/p24-mobile-lib-android-example](https://github.com/przelewy24/p24-mobile-lib-android-example)

## 1. Project configuration

The first step is to set the value `minSdkVersion=19` in file `build.gradle`

### Adding dependencies

In the Android Studio environment, it is possible to add a library module by using the command: „File → New → New module...”. From the list „New module” select „Import .JAR or .AAR Package” and click „Next”. In the field „File name” provide access path to file `p24Lib.aar`. As a „Subproject name”, provide „p24Lib” and click „Finish”.

The next step is to add a dependency to the created library module by modifying the file `build.gradle` and placing the following entry in the section „dependencies”:

`compile project(':p24lib')`

Since the library uses the AndroidX library, the following dependency must be added:

`implementation 'androidx.appcompat:appcompat:+'`

When using SafetyNet you need add:

`implementation 'com.google.android.gms:play-services-wallet:16.0.1'`

When integrating with GooglePlay you need add:

`implementation 'com.google.android.gms:play-services-safetynet:+'`

Below is an example of a „dependencies” section:

```gradle

dependencies {
	//other dependencies
    implementation project(':p24Lib')
    implementation 'com.google.android.gms:play-services-wallet:16.0.1' //necessary if google pay used
    implementation 'com.google.android.gms:play-services-safetynet:+' //necessary if safetynet function enabled
    implementation 'androidx.appcompat:appcompat:1.1.0'
}

```

### Proguard

If the target project don't use GooglePlay, the following entry should be added to the proguard file:

```proguard
-dontwarn com.google.android.gms.wallet.**
```

### Definition of AndroidManifest file

Add the following to `AndroidManifest.xml` file:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

Next, in `application` section, add `TransferActivity`:

```xml
<activity android:name=„pl.przelewy24.p24lib.transfer.TransferActivity"
          android:configChanges="orientation|keyboard|keyboardHidden|screenSize"
          android:theme="@style/Theme.AppCompat.Light.DarkActionBar”/>
```

__All Activities in the library draw on the AppCompatActivity, which is why  „Theme.AppCompat.*” group styles as well as derivative styles should be used__

In the case of default Activity settings, the WebView will get reloaded during the rotation of the library display screen, which may cause a return from the bank’s website to the list of payment forms and render further transaction processing impossible. In order to prevent the reloading of the library window, the following parameter must be set:

```xml
android:configChanges="orientation|keyboard|keyboardHidden|screenSize"
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

    <application >

		<!--other activities-->

        <activity android:name="pl.przelewy24.p24lib.transfer.TransferActivity"
                  android:configChanges="keyboardHidden|orientation|keyboard|screenSize"
                  android:theme="@style/Theme.AppCompat.Light.DarkActionBar"/>

    </application>

</manifest>


```

## 2. SSL Pinning

The library has a Pinning SSL mechanism that can be activated globally for webview calls.
If you want use this feature, please make sure configuration is setup before any library methods calls. Example:

```java
SdkConfig.setCertificatePinningEnabled(true);
```

**UWAGA!!**

 > When activating SSL Pinning, keep in mind that the certificates embedded in the library have their validity time. Before time of their expiry, Przelewy24 will be sending out appropriate information and updating.

## 3. Split payment

The function is available for transfer calls (trnRequest, trnDirect, express). To activate, use the appropriate flag before the transaction request:

```java
SdkConfig.setSplitPaymentEnabled(true);
```

## 4. trnDirect transaction call

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

With the configurational objects complete, one may proceed to call Activity for a transaction. The initiation looks as follows:

```java
Intent intent = TransferActivity.getIntentForTrnDirect(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

In order to serve the transaction result, one must override `Activity.onActivityResult`:

```java
@Override
protected void onActivityResult(int reqCode, int resCode, Intent data) {
    super.onActivityResult(reqCode, resCode, data);
    if (reqCode == TRANSACTION_REQUEST_CODE) {
        if (resCode == RESULT_OK) {
            TransferResult result = TransferActivity.parseResult(data);

            if (result.isSuccess()) {
                // success
            } else {
                //error
                String errorCode = result.getErrorCode();
            }
        } else {
            //cancel
        }
    }
}
```
`TransferActivity` returns only information regarding the completion of the transaction. It need not mean that the transaction has been verified by the partner’s server. That is why, each time the `isSuccess()` status is obtained, the application should call its own backend to check the transaction status.


## 5. trnRequest transaction call

During the registration with the "trnRegister" method, additional parameters should be provided:
- `p24_mobile_lib=1`
- `p24_sdk_version=X` – where X is a moibile lib version provided by `P24SdkVersion.value()` method

This parameters  allows Przelewy24 to classify the transaction as a mobile transaction. A Token registered without this parameter will not work in the mobile application (an error will appear upon return to the bank and the library file will not detect payment completion).

**NOTE!**

 > When registering a transaction which is to be carried out in a mobile library, remember about the additional parameters:
- `p24_channel` – unless set, the library will feature the payment options „traditional transfer” and „use prepayment”, which are unnecessary in case of mobile payments. In order to deactivate them, use flags that disregard these forms (e.g. value 3 – payments and cards, default entry setting, directly with parameters)
- `p24_method` – if a given transaction in the library is to have a specific, preset method of payment, this method must be selected during the registration
- `p24_url_status` - the address to be used for transaction verification by the partner’s server once the payment process in the mobile library is finished

The transaction parameters must be set using the token of a transaction registered earlier. Optionally, the sandbox server and bank configuration may be set:

```java
TrnRequestParams params = TrnRequestParams
                      .create("XXXXXXXXXX-XXXXXX-XXXXXX-XXXXXXXXXX")
                      .setSandbox(true);
```

Next, create `Intent` to call transaction `Activity` and run it:

```java
Intent intent = TransferActivity.getIntentForTrnRequest(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

The transaction result should be served in the same way as in the case of "trnDirect".

## 6. Express transaction call

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

## 7. Passage 2.0 transaction call

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

## 8. Google Pay

The data flow process using this payment method looks as follows:

![](img/diagram_google_pay_eng.png)

To use the Google Pay payment you must first make an additional configuration of the project:

At the 'application' node, please add the 'GooglePayActivity' activity:

```xml
<activity
    android:name="pl.przelewy24.p24lib.google_pay.GooglePayActivity"
    android:theme="@style/Theme.AppCompat.Translucent"
    android:configChanges="keyboardHidden|orientation|keyboard|screenSize">
</activity>
```

and place the appropriate activation entry for the Google Pay service:

```xml
<meta-data
    android:name="com.google.android.gms.wallet.api.enabled"
    android:value="true" />
```

Then, in the `build.gradle` file, you must add a dependency for the Google library:

`implementation 'com.google.android.gms:play-services-wallet:16.+'`


To initiate a transaction, you must pass the transaction parameters and the GooglePayTransactionRegistrar object that is used to register the transaction:

```java
GooglePayParams params = GooglePayParams.create(MERCHANT_ID, getItemPrice(), "PLN")
				.setSandbox(IS_SANDBOX);

Intent intent = GooglePayActivity.getStartIntent(this, params, getGooglePayTrnRegistrar());
startActivityForResult(intent, GOOGLE_PAY_REQUEST_CODE);
```

The GooglePayTransactionRegistrar interface allows you to implement the exchange of the token received from Google Pay into the P24 transaction token. When the `register` method is called, communicate with the P24 servers, pass the Google Pay payment token as the `p24_method_ref_id` parameter, and then pass the transaction token to the library using the callback method by calling the `onTransactionRegistered` method.

```java
private GooglePayTransactionRegistrar getGooglePayTrnRegistrar() {
    return new GooglePayTransactionRegistrar() {
        @Override
        public void register(String methodRefId, GooglePayTransactionRegistrarCallback callback) {
            // register transaction and retreive token
            callback.onTransactionRegistered("P24_TRANSACTION_TOKEN");
        }
    };
}
```

Result handling looks as follows:

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);

    if (requestCode == GOOGLE_PAY_REQUEST_CODE) {
        if (resultCode == RESULT_OK) {
            GooglePayResult result = GooglePayActivity.parseResult(data);
            if (result.isError())
                showError("Google Pay error. Code: " + result.getErrorCode());

            if (result.isCompleted())
                showSuccess("Google Pay completed");
        } else {
            showCancel("Google Pay canceled");
        }
	} 
}
```

For more information how to handle communication with P24 servers visit here: [https://docs.przelewy24.pl/Google_Pay](https://docs.przelewy24.pl/Google_Pay).
