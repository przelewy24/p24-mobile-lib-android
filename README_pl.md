# Dokumentacja biblioteki Przelewy24 - Android

Ogólne informacje o działaniu bibliotek mobilnych w systemie Przelewy24 znajdziesz pod adresem:

- [https://github.com/przelewy24/p24-mobile-lib-doc](https://github.com/przelewy24/p24-mobile-lib-doc)

Przykład implementacji biblioteki:

- [https://github.com/przelewy24/p24-mobile-lib-android-example](https://github.com/przelewy24/p24-mobile-lib-android-example)

## 1. Konfiguracja projektu

Konfigurację należy rozpocząć od ustawienia wartości `minSdkVersion=14` w pliku `build.gradle`

### Dodawanie zależności

W środowisku Android Studio możliwe jest doadnie modułu biblioteki poprzez polecenie: „File → New → New module...”. Z listy „New module” wybrać „Import .JAR or .AAR Package” i kliknąć „Next”. W polu „File name” podać ścieżkę do pliku „p24Lib.aar”, jako „Subproject name” podać „p24Lib” i kliknąć „Finish”.

Kolejny krok to dodanie zależności do stworzonego modułu biblioteki poprzez modyfikację pliku `build.gradle` i umieszczenie w sekcji „dependencies” wpisu:

`compile project(':p24lib')`

Biblioteka wykorzystuje bibliotekę AppCompat v7, dlatego należy różnież dodać zależność:

`compile 'com.android.support:appcompat-v7:26.+'`

Przykładowo sekcja „dependencies” powinna wyglądać tak:

```gradle

dependencies {
	//other dependencies
    compile 'com.android.support:appcompat-v7:26.+'
    compile project(':p24Lib')
}

```

### Definiowanie pliku AndroidManifest

Do pliku `AndroidManifest.xml`, w węźle `manifest` dodać:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

W sytuacji kiedy wykorzystujemy funkcję wklejania kodu SMS należy również dodać:

```xml
<uses-permission android:name="android.permission.RECEIVE_SMS"/>
```

Następnie w węźle `application` dodać aktywność `TransferActivity`:

```xml
<activity android:name=„pl.przelewy24.p24lib.transfer.TransferActivity"
          android:configChanges="orientation|keyboard|keyboardHidden"
          android:theme="@style/Theme.AppCompat.Light.DarkActionBar”/>
```

Oraz aktywność `PaymentSettingsActivity`:

```xml
<activity android:name="pl.przelewy24.p24lib.settings.PaymentSettingsActivity"
          android:configChanges="orientation|keyboard|keyboardHidden"
          android:theme="@style/Theme.AppCompat.Light.DarkActionBar"/>
```

__Wszystkie Activity w bibliotece dziedziczą po AppCompatActivity, dlatego należy do nich stosować style z grupy „Theme.AppCompat.*” i pochodne__

Przy domyślnych ustawieniach Activity podczas obrotu ekranu biblioteki nastąpi przeładowanie WebView, co może powodować powrót ze strony banku do listy form płatności i uniemożliwić sfinalizowanie transakcji. Aby okno biblioteki nie przeładowywało się konieczne jest ustawienie parametru:

```xml
android:configChanges="orientation|keyboard|keyboardHidden"
```
Przykładowo plik `AndroidManifext.xml` powinien wyglądać tak:

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

## 2. Wywołanie transakcji trnDirect

W tym celu należy ustawić parametry transakcji korzystając z klasy buildera, podając Merchant Id i klucz do CRC:

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

Parametry opcjonalne:

```java
builder.urlStatus("https://XXXXXX")
       .method(25)
       .timeLimit(90)
       .channel(1)
       .transferLabel("transfer label")
       .shipping(0);
```
Następnie stworzyć obiekt z parametrami wywołania transakcji, odpowiedni dla danej metody:

```java
TrnDirectParams params = TrnDirectParams.create(transactionParams);
```

Opcjonalne można ustawić wywołanie transakcji na serwer Sandbox:

```java
params.setSandbox(true);
```

Również opcjonalne można dodać ustawienia zachowania biblioteki dla stron banków (style mobile na stronach banków – domyślnie włączone, czy biblioteka ma zapamiętywać logi i hasło do banków, czy biblioteka ma automatycznie przeklejać hasła sms do formularza potwierdzenia transakcji w banku):

```java
SettingsParams settingsParams = new SettingsParams();
settingsParams.setEnableBanksRwd(true);
settingsParams.setSaveBankCredentials(true);
settingsParams.setReadSmsPasswords(true);
params.setSettingsParams(settingsParams);
```

W przypadku ustawienia `setReadSmsPasswords` na `true` należy również dodać do manifestu:

```xml
<uses-permission android:name="android.permission.RECEIVE_SMS"/>
```

Mając gotowe obiekty konfiguracyjne możemy przystąpić do wywołania `Activity` dla transakcji. Uruchomienie wygląda następująco:

```java
Intent intent = TransferActivity.getIntentForTrnDirect(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

Aby obsłużyć rezultat transakcji należy rozszerzyć metodę `Activity.onActivityResult`:

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
`TransferActivity` zwraca tylko informację o tym, że transakcja się zakończyła. Nie zawsze oznacza to czy transakcja jest zweryfikowana przez serwer partnera, dlatego za każdym razem po uzyskaniu statusu `isSuccess()` aplikacja powinna odpytać własny backend o status transakcji.

## 3. Wywołanie transakcji trnRequest

Podczas rejestracji transakcji metodą "trnRegister" należy podać parametr `p24_mobile_lib=1`, dzięki czemu system Przelewy24 będzie wiedział że powinien traktować transakcję jako mobilną. Token zarejestrowany bez tego parametru nie zadziała w bibliotece mobilnej (wystąpi błąd po powrocie z banku i okno biblioteki nie wykryje zakończenia płatności).

**UWAGA!**

 > Rejestrując transakcję, która będzie wykonana w bibliotece mobilnej należy
pamiętać o dodatkowych parametrach:
- `p24_channel` – jeżeli nie będzie ustawiony, to domyślnie w bibliotece pojawią się
formy płatności „przelew tradycyjny” i „użyj przedpłatę”, które są niepotrzebne przy płatności mobilnej. Aby wyłączyć te opcje należy ustawić w tym parametrze flagi nie
uwzględniające tych form (np. wartość 3 – przelewy i karty, domyślnie ustawione w
bibliotece przy wejściu bezpośrednio z parametrami)
- `p24_method` – jeżeli w bibliotece dla danej transakcji ma być ustawiona domyślnie
dana metoda płatności, należy ustawić ją w tym parametrze przy rejestracji
- `p24_url_status` - adres, który zostanie wykorzystany do weryfikacji transakcji przez serwer partnera po zakończeniu procesu płatności w bibliotece mobilnej

Należy ustawić parametry transakcji podając token zarejestrowanej wcześniej transakcji, opcjonalnie można ustawić serwer sandbox oraz konfigurację banków:

```java
TrnRequestParams params = TrnRequestParams
                      .create("XXXXXXXXXX-XXXXXX-XXXXXX-XXXXXXXXXX")
                      .setSandbox(true)
                      .setSettingsParams(settingsParams);
```

Następnie należy stworzyć `Intent` do wywołania `Activity` transakcji i uruchomić go:

```java
Intent intent = TransferActivity.getIntentForTrnRequest(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

Rezultat transakcji należy obsłużyć identycznie jak dla wywołania "trnDirect".

## 4. Wywołanie transakcji Ekspres

Należy ustawić parametry transakcji podając url uzyskany podczas rejestracji transakcji w systemie Ekspres. Transakcja musi być zarejestrowana jako mobilna.

```java
ExpressParams params = ExpressParams.create(expresTransactionUrl);
```

Następnie należy stworzyć Intent do wywołania Activity transakcji i uruchomić go:

```java
Intent intent = TransferActivity.getIntentForExpress(getApplicationContext(), params);
activity.startActivityForResult(intent, TRANSACTION_REQUEST_CODE);
```

Rezultat transakcji należy obsłużyć identycznie jak dla wywołania "trnDirect".

## 5. Wywołanie transakcji z Pasażem 2.0

Należy ustawić parametry transakcji identycznie jak dla wywołania "trnDirect", dodając odpowiednio przygotowany obiekt koszyka:

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

Wywołanie transakcji oraz parsowanie wyniku jest realizowane identycznie jak dla wywołania "trnDirect".
