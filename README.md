# speech-to-text-python


```python
!pip install azure-cognitiveservices-speech

import azure.cognitiveservices.speech as speechsdk

speech_config = speechsdk.SpeechConfig(subscription="<YOUR_SPEECH_SERVICE_KEY>", region="westeurope")

import logging
import os
import sys
import requests
import time
import swagger_client as cris_client

logging.basicConfig(stream=sys.stdout, level=logging.DEBUG,
        format="%(asctime)s %(message)s", datefmt="%m/%d/%Y %I:%M:%S %p %Z")

# Your subscription key and region for the speech service
SUBSCRIPTION_KEY = "<YOUR_SPEECH_SERVICE_KEY>"
SERVICE_REGION = "westeurope"

NAME = "Simple transcription"
DESCRIPTION = "Simple transcription description"

LOCALE = "tr-TR"
RECORDINGS_BLOB_URI = "https://<YOUR_BLOB_STORAGE_ACCOUNT>.blob.core.windows.net/demo?sp=racwdli&st=2022-03-31T07:25:40Z&se=2022-04-07T15:25:40Z&spr=https&sv=2020-08-04&sr=c&sig=qqZZw%2Fyqs83XNSbUJKQ54AqR%2FSiG3ZZ6XlXT1qqILPM%3D"

# Provide the uri of a container with audio files for transcribing all of them with a single request
RECORDINGS_CONTAINER_URI = "https://<YOUR_BLOB_STORAGE_ACCOUNT>.blob.core.windows.net/demo?sp=racwdli&st=2022-03-31T07:25:40Z&se=2022-04-07T15:25:40Z&spr=https&sv=2020-08-04&sr=c&sig=qqZZw%2Fyqs83XNSbUJKQ54AqR%2FSiG3ZZ6XlXT1qqILPM%3D"
# Set model information when doing transcription with custom models
MODEL_REFERENCE = None  # guid of a custom model


def transcribe_from_single_blob(uri, properties):
    """
    Transcribe a single audio file located at `uri` using the settings specified in `properties`
    using the base model for the specified locale.
    """
    transcription_definition = cris_client.Transcription(
        display_name=NAME,
        description=DESCRIPTION,
        locale=LOCALE,
        content_urls=[uri],
        properties=properties
    )

    return transcription_definition


def transcribe_with_custom_model(api, uri, properties):
    """
    Transcribe a single audio file located at `uri` using the settings specified in `properties`
    using the base model for the specified locale.
    """
    # Model information (ADAPTED_ACOUSTIC_ID and ADAPTED_LANGUAGE_ID) must be set above.
    if MODEL_REFERENCE is None:
        logging.error("Custom model ids must be set when using custom models")
        sys.exit()

    model = api.get_model(MODEL_REFERENCE)

    transcription_definition = cris_client.Transcription(
        display_name=NAME,
        description=DESCRIPTION,
        locale=LOCALE,
        content_urls=[uri],
        model=model,
        properties=properties
    )

    return transcription_definition


def transcribe_from_container(uri, properties):
    """
    Transcribe all files in the container located at `uri` using the settings specified in `properties`
    using the base model for the specified locale.
    """
    transcription_definition = cris_client.Transcription(
        display_name=NAME,
        description=DESCRIPTION,
        locale=LOCALE,
        content_container_url=uri,
        properties=properties
    )

    return transcription_definition


def _paginate(api, paginated_object):
    """
    The autogenerated client does not support pagination. This function returns a generator over
    all items of the array that the paginated object `paginated_object` is part of.
    """
    yield from paginated_object.values
    typename = type(paginated_object).__name__
    auth_settings = ["apiKeyHeader", "apiKeyQuery"]
    while paginated_object.next_link:
        link = paginated_object.next_link[len(api.api_client.configuration.host):]
        paginated_object, status, headers = api.api_client.call_api(link, "GET",
            response_type=typename, auth_settings=auth_settings)

        if status == 200:
            yield from paginated_object.values
        else:
            raise Exception(f"could not receive paginated data: status {status}")


def delete_all_transcriptions(api):
    """
    Delete all transcriptions associated with your speech resource.
    """
    logging.info("Deleting all existing completed transcriptions.")

    # get all transcriptions for the subscription
    transcriptions = list(_paginate(api, api.get_transcriptions()))

    # Delete all pre-existing completed transcriptions.
    # If transcriptions are still running or not started, they will not be deleted.
    for transcription in transcriptions:
        transcription_id = transcription._self.split('/')[-1]
        logging.debug(f"Deleting transcription with id {transcription_id}")
        try:
            api.delete_transcription(transcription_id)
        except cris_client.rest.ApiException as exc:
            logging.error(f"Could not delete transcription {transcription_id}: {exc}")


def transcribe():
    logging.info("Starting transcription client...")

    # configure API key authorization: subscription_key
    configuration = cris_client.Configuration()
    configuration.api_key["Ocp-Apim-Subscription-Key"] = SUBSCRIPTION_KEY
    configuration.host = f"https://{SERVICE_REGION}.api.cognitive.microsoft.com/speechtotext/v3.0"
    
    
    configuration.debug = True

    # create the client object and authenticate
    client = cris_client.ApiClient(configuration)

    # create an instance of the transcription api class
    api = cris_client.DefaultApi(api_client=client)

    # Specify transcription properties by passing a dict to the properties parameter. See
    # https://docs.microsoft.com/azure/cognitive-services/speech-service/batch-transcription#configuration-properties
    # for supported parameters.
    properties = {
        # "punctuationMode": "DictatedAndAutomatic",
        # "profanityFilterMode": "Masked",
        # "wordLevelTimestampsEnabled": False,
         "diarizationEnabled": True,
         "destinationContainerUrl": "https://<YOUR_BLOB_STORAGE_ACCOUNT>.blob.core.windows.net/demo?sp=racwdli&st=2022-03-31T07:25:40Z&se=2022-04-07T15:25:40Z&spr=https&sv=2020-08-04&sr=c&sig=qqZZw%2Fyqs83XNSbUJKQ54AqR%2FSiG3ZZ6XlXT1qqILPM%3D",
        # "timeToLive": "PT1H"
    }

    # Use base models for transcription. Comment this block if you are using a custom model.
    # transcription_definition = transcribe_from_single_blob(RECORDINGS_BLOB_URI, properties)

    # Uncomment this block to use custom models for transcription.
    # transcription_definition = transcribe_with_custom_model(api, RECORDINGS_BLOB_URI, properties)

    # Uncomment this block to transcribe all files from a container.
    transcription_definition = transcribe_from_container(RECORDINGS_CONTAINER_URI, properties)

    created_transcription, status, headers = api.create_transcription_with_http_info(transcription=transcription_definition)

    # get the transcription Id from the location URI
    transcription_id = headers["location"].split("/")[-1]

    # Log information about the created transcription. If you should ask for support, please
    # include this information.
    logging.info(f"Created new transcription with id '{transcription_id}' in region {SERVICE_REGION}")

    logging.info("Checking status.")

    completed = False

    while not completed:
        # wait for 5 seconds before refreshing the transcription status
        time.sleep(5)

        transcription = api.get_transcription(transcription_id)
        logging.info(f"Transcriptions status: {transcription.status}")

        if transcription.status in ("Failed", "Succeeded"):
            completed = True

        if transcription.status == "Succeeded":
            pag_files = api.get_transcription_files(transcription_id)
            for file_data in _paginate(api, pag_files):
                if file_data.kind != "Transcription":
                    continue

                audiofilename = file_data.name
                results_url = file_data.links.content_url
                results = requests.get(results_url)
                logging.info(f"Results for {audiofilename}:\n{results.content.decode('utf-8')}")
                
                # Write results to a file
                fileName = os.path.basename(audiofilename);
                f = open(f"{fileName}.transcription.txt", "w")
                f.write(f"{results.content.decode('utf-8')}")
                f.close()
                
        elif transcription.status == "Failed":
            logging.info(f"Transcription failed: {transcription.properties.error.message}")


if __name__ == "__main__":
    transcribe()
```
