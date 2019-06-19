# Laravel Socialite mit Google Fitness API

Als ich mir vor etwa drei Wochen ein Fitnessarmband gekauft habe, war die Absicht meine sportlichen Aktivitäten zu erhöhen. Gesagt getan: Das Xiaomi Band3 gekauft, ein Wochenende in Berlin verbracht und mich solange bewegt bis die Füße geblutet haben - natürlich nur im übertragenen Sinne. 

Von Anfang an war mir klar, dass ich früher oder später meine Fitnessdaten auf dem PC aufwerten möchte. Leider bieten die wenigsten Anbieter dieser Fitnessarmbänder eine direkte Softwarelösung um seine Daten automatisiert exportieren und anderweitig auswerten zu können.

Glücklicherweise ist eine Synchronisation zu Google Fit inzwischen nicht mehr unüblich. Dort lassen sich die Daten für den Anwender zwar auch nur über eine App analysieren, dafür bietet Google zumindest eine Rest API an.

Um mir beim Auslesen der notwendigen Fitnessdaten über die Rest API die Arbeit möglichst einfach zu gestalten, setze ich für die Authentifizierung [Laravel Socialite](https://laravel.com/docs/5.8/socialite) ein. Für die restliche Kommunikation setze ich auf den [offiziellen PHP Client](https://github.com/googleapis/google-api-php-client) von Google. 

In diesem Beitrag werde ich mich überwiegend auf die Abfrage von Fitnessdaten fokussieren. Eine Anleitung für den Umgang mit Socialite und dem PHP Client [gibt es hier](https://laravel-news.com/google-api-socialite).

Im ersten Schritt wird über Socialite ein Token zur Authentifizierung generiert und an den Google Client übergeben.

```php
$user = Socialite::driver('google')->user();

$client = new \Google_Client();
$client->setApplicationName("Laravel");
$client->setDeveloperKey(env('GOOGLE_SERVER_KEY'));

$google_client_token = [
    'access_token' => $user->token,
    'refresh_token' => $user->refreshToken,
    'expires_in' => $user->expiresIn
];
$client->setAccessToken(json_encode($google_client_token));
```

Als nächstes bauen wir uns für die Abfrage der API die notwendigen Objekte zusammen. Da für die Abfragen jeweils immer die passenden PHP Klassen zur Verfügung stehen, sind mehrdimensionale Arrays oder JSON Strings nicht erforderlich. 

Für die Abfrage meiner Fitnessdaten möchte ich in diesem Fall die Anzahl meiner geleisteten Schritte, und die zurückgelegte Distranz innerhalb eines vorgegebenen Zeitraums wissen. Hierzu greife ich auf die [Datentypen](https://developers.google.com/fit/android/data-types) *com.google.step_count.delta* (Schrittzähler) bzw. *com.google.distance.delta* (Distanz) zurück. 

```php
$aggregateBy1 = new Google_Service_Fitness_AggregateBy();
$aggregateBy1->dataTypeName = 'com.google.step_count.delta';
$aggregateBy1->dataSourceId = 'derived:com.google.step_count.delta:com.google.android.gms:estimated_steps';

$aggregateBy2 = new Google_Service_Fitness_AggregateBy();
$aggregateBy2->dataTypeName = 'com.google.distance.delta';
$aggregateBy2->dataSourceId = 'derived:com.google.distance.delta:com.google.android.gms:merge_distance_delta';

$postBody = new Google_Service_Fitness_AggregateRequest();
$postBody->setAggregateBy([$aggregateBy1, $aggregateBy2]);
```
Als nächstes nehmen wir uns den Wert [BucketByTime](https://developers.google.com/fit/rest/v1/reference/users/dataset/aggregate#bucketByTime) vor. Dieser ist für den zeitlichen Intervall der abgefragten Daten zuständig, in dem diese später bei der Ausgabe angeordnet werden sollen. Der Wert im Beispiel entspricht einen Tag in Millisekunden. 
```php
$bucketByTime = new \Google_Service_Fitness_BucketByTime();
$bucketByTime->durationMillis = 86400000;

$postBody->setBucketByTime($bucketByTime);
```
Nun legen wir noch den Zeitraum fest indem die Daten abgefragt werden sollen. Im Beispiel sind dies etwa die letzten 7 Tage, natürlich wieder für die API in Millisekunden gerechnet.
```php
$startDateTime = new \DateTime();
$startDateTime->sub(new \DateInterval('P7D'));

$endDateTime = new \DateTime();
$endDateTime->setTime(23,59,59,999);

$postBody->setStartTimeMillis($startDateTime->getTimestamp()*1000);
$postBody->setEndTimeMillis($endDateTime->getTimestamp()*1000);
```
Zuletzt wird nun eine Instanz von dem Google Fitness Service erstellt und das erstellte Abfragenobjekt übergeben. Für $userId können wir in diesem Fall *me* als Wert übergeben, sofern man auf die Daten des mit dem Google Client authentifizierten Benutzers zugreifen möchte.   
```php
$service = new \Google_Service_Fitness($client);
$response = $service->users_dataset->aggregate($userId, $postBody);
```

Als Ausgabe sollte nun ein großes Objekt mit den gewünschten Informationen zurückgegeben wird. Aufgrund der Größe der Rückgabeinformationen ist es ratsam nicht sofort alle Daten auf einmal auszulesen, sofern man den Intervall wie in meinem Beispiel auf einen täglichen Wert setzt.

```
object(Google_Service_Fitness_DataPoint)[345]
  protected 'collection_key' => string 'value' (length=5)
  public 'computationTimeMillis' => null
  public 'dataTypeName' => string 'com.google.distance.delta' (length=25)
  public 'endTimeNanos' => string '1560630000000000000' (length=19)
  public 'modifiedTimeMillis' => null
  public 'originDataSourceId' => string 'raw:com.google.distance.delta:com.xiaomi.hm.health:' (length=51)
  public 'rawTimestampNanos' => null
  public 'startTimeNanos' => string '1560543830000000000' (length=19)
  protected 'valueType' => string 'Google_Service_Fitness_Value' (length=28)
  protected 'valueDataType' => string 'array' (length=5)
  protected 'internal_gapi_mappings' => 
    array (size=0)
      empty
  protected 'modelData' => 
    array (size=0)
      empty
  protected 'processed' => 
    array (size=0)
      empty
  public 'value' => 
    array (size=1)
      0 => 
        object(Google_Service_Fitness_Value)[346]
          protected 'collection_key' => string 'mapVal' (length=6)
          public 'fpVal' => float 2622.5818326596
          public 'intVal' => null
          protected 'mapValType' => string 'Google_Service_Fitness_ValueMapValEntry' (length=39)
          protected 'mapValDataType' => string 'array' (length=5)
          public 'stringVal' => null
          protected 'internal_gapi_mappings' => 
            array (size=0)
              empty
          protected 'modelData' => 
            array (size=0)
              empty
          protected 'processed' => 
            array (size=0)
              empty
          public 'mapVal' => 
            array (size=0)
              empty
```

Dieses Codebeispiel werde ich später zusammen mit einem Tool zur Auswertung der Google Fitnessdaten auf [Github](https://github.com/rotfuchs) unter freier Lizenz zur Verfügung stellen. Geplant ist eine Veröffntlichung im Laufe des Jahres 2019.
