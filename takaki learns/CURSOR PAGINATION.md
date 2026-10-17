---
type: concept
---
 ## set cursor:
 Initial => cursor = none
 fetch data (limit = 30), limit + 1
 if fetchedData.length > limit => there are more data
 if olderData:
     curosr = { createdAt: fetchedMessages.slice(0, limit)[length - 1], id: this data.id }
else: 
     cursor = {}

encode this cursor so that it becomes string
send this cursor along side with fetched data to frontend
=> front will have this cursor and send it whenver it needs to load the new data

