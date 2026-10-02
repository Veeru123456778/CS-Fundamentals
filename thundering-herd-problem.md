# Thundering Herd Problem

If we have a bunch of request that are coming to my machine and if large number of request failed due to server being overwhelmed (now server is unable to process the request now if we retry immediately then also they will got failed)

So Simple Solution:
Add a randome jitter (value) in the retry time to get the system up again and process things again.
Dont make the microservice for everything for the sake of making it.