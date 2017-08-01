# Queue Constructor
___
### Default Options
```javascript
const Queue = require('rethinkdb-job-queue')
const q = new Queue()
```
___
### Queue Connection

Option  | Type | Default
:---: | :---: | :---: 
host | String | localhost
port | Integer | 28015
db | String | rjqJobQueue
___
### Connection Driver
```javascript
const rethinkdbdash = require('rethinkdbdash')
const Queue = require('rethinkdb-job-queue')

const db01driver = rethinkdbdash({ host: 'db01.domain.com', db: 'JobQueue' })

const q = new Queue(db01driver)
```
___
### Queue Options
Option | Type | Default | Version
:---: | :---: | :---: | :---:
name | String | rjqJobList  
databaseInitDelay | Integer | 1000 ms (1 sec) 
queryRunOptions | Boolean | { readMode: 'majority' }  
changeFeed | Boolean | true
concurrency | Integer | 1 
masterInterval | Integer | 310000 ms (5 min 10 sec) 
limitJobLogs | Integer | 1000 log entries | v3.1.0
removeFinishedJobs | Bool / Int | 15552000000 ms (180 days)

#### Queue databaseInitDelay Option
If ***multiple*** Queue objects are instantiated at the ***same time*** it might cause two databases or tables with the ***same name*** to be created within the same RethinkDB instance.
* To mitigate this issue there is a delay before the databases resources are created. 
* This delay comprises of a fixed portion and a random portion.
* The ___databaseInitDelay___ option is exposing the ___fixed portion___ of the initialization delay.
* The ___random portion is hard coded___ and will be between zero and one second.

#### Queue queryRunOptions Option
If you do change the run option ___readMode___ to ___single___ and you experience the same job being processed by multiple Queue workers, change it back to the default majority value.

#### Queue changeFeed Option
If enabled the changeFeed option will cause the Queue object to raise events for the entire queue rather than just the local Queue object.
Once the changes are received by the Queue object, they are checked to ensure they are not local changes.
If the changes are detected as local changes (a change made by self) they are ignored because the events will be raised by the Queue object.
If the changes are not local then the relevant event is raised.

**Local Event Example**
```javascript
const Queue = require('rethinkdb-job-queue')

const qOptions = {
  changeFeed: false
}

const q = new Queue(null, qOptions)

q.on('completed', (queueId, jobId, isRepeating) => {
  console.log('Job completed: ' + jobId)
})
```

**Global Event Example**
```javascript
const Queue = require('rethinkdb-job-queue')

const qOptions = {
  changeFeed: true // This is the default and is not required
}

const q = new Queue(null, qOptions)

q.on('completed', (queueId, jobId, isRepeating) => {
  console.log('Job completed: ' + jobId)
})
```

#### Queue masterInterval Option
The ___masterInterval___ option determines how often the ___Queue Master___ review process is carried out against the queue.
**`Warning:`** Setting the masterInterval value too low will cause excessive database load. Every time this period lapses queries are run against the database. Keep this value as high as possible.

##### Queue Master

The Queue Master role in rethinkdb-job-queue is an integral role to __ensure delayed and failed jobs get processed and the database is cleaned__.
When the ___time period elapses___, the Queue Master will ___review___ the database table backing the queue. This is called the __Queue Review process__.

A Queue Master will perform four tasks within the job queue during the Queue Review process:
* ***Failed Node.js Process***
Discover and enable jobs that have failed due to the Node.js process crashing or hanging.
* ***Remove Finished Jobs***
Remove completed, cancelled, or terminated jobs from the queue.
* ***Delayed Job Processing***
Enable processing of delayed jobs or failed jobs waiting for retry.
* ***Update Queue State***
At the completion of the review process the queue State Document will be updated.

###### Failed Node.js Process
During normal queue operation, Queue objects processing jobs will ___detect when a job has taken too long and is operating past its timeout value.___ If this situation occurs the job status in the database is set to failed and the job will be delayed based on the ___retryDelay___, ___retryCount___, and ___retryMax___ values.

However, if a Node.js process ___fails for any reason___ whilst working on a job, ___the job will not be completed and will remain in the database with an active status causing an orphaned job___.

To ensure the job is not forgotten, ___a Queue Master will repeatedly review the queue database backing table based on the masterInterval.___ When the Queue Master reviews the queue backing table, it looks for jobs that have a status of active and are past their ___dateEnable___ value. __The _dateEnable_ value is set when the job is created or when it is retrieved from the database for processing.__

__The queue review process will update the job status based on the _retryCount_ and _retryMax_ values__

`NOTE:`If the jobs ___retryCount___ value is less than the ___retryMax___ then the job status will be set to ___'failed'___ and the ___retryCount___ value will be incremented. This job will now be ___ready for processing.___

`NOTE:`If the jobs ___retryCount___ value is equal to the ___retryMax___ value then the job status will be set to terminated and the job is considered finished.

>It is possible for normal job being processed to extend past its initial timeout value and be marked as failed by the Queue Master review process. To prevent this, call the Job.progress method on the Job object. When progress for a job is updated, the dateEnable value and the timeout process also get updated. Therefore calling Job.progress periodically within the job timeout period will prevent the job from erroneously being marked as failed on review.

###### Remove Finished Jobs

In this context a ___finished job___ is defined as a job in the queue that has a status of either ___completed, cancelled, or terminated.___

`NOTE:`Once a job has finished processing it will no longer be an active part of the queue. The job details in the database including its log entries and other properties are just taking up space.

Now if you are processing thousands of jobs a day this might not be a big deal and you may very well be happy to just leave the job details in the database for future reference. However if you are processing millions of jobs a day, the space taken up by the completed jobs could add up over a year or more. If that is the case then you will want to remove finished jobs from the database to free up space.

Fortunately Queue objects have three options for cleaning up jobs once they are finished based on the ___removeFinishedJobs___ Queue Option.

Two of the values you can set the ___removeFinishedJobs___ Queue option to will be ignored by the Queue Master review process:  ___true or false.___

If you set the ___removeFinishedJobs___ option to ___true___, finished jobs will be ___removed from the database immediately.___

If you set the ___removeFinishedJobs___ option to ___false___, jobs will ___never be removed from the database___ no matter what their status is.

The third value you can assign to the ___removeFinishedJobs___ Queue option is a ___positive Integer___. This number ___represents a time period in milliseconds.___

Jobs will be considered ___eligible to be removed___ when their ___dateFinished property is older than the dateFinished plus removeFinishedJobs resultant date.___

The Queue Master review process will ___permanently remove these jobs from the queue.___

Setting the ___removeFinishedJobs___ value to a ___low number___ such as 7 days (in milliseconds) would give you enough time to use the job logs to help you ___debug issues___ while still keeping your queue database clean.

Alternatively, setting ___removeFinishedJobs___ value to a ___high number___ such as 365 days (in milliseconds) would give you plenty of data ___for analysis___.

`NOTE:` Please consider disabling the ___removeFinishedJobs___ process if you can. It can always be enabled at a later date.

###### Delayed Job Processing

`Important:` The following is only valid if the Queue Master Queue object has a process handler assigned. If it does not, the Update Queue State task below will enable delayed job processing.

In a busy queue the database will be queried upon completion of jobs in order to find more jobs that need processing. This includes ___finding jobs with a status of waiting or failed___ with the current date after the job ___dateEnable___ value.

`NOTE:`If the last job in the queue ___fails___ and the ___retryDelay___ value is ___not 0___, the job will be delayed for retry and the queue will enter an idle state. There may be other jobs delayed in the queue also.

Without something initiating the queue to process jobs, the last job will remain in the database until more jobs are added to the queue.

To prevent this situation from delaying the last job well beyond its dateEnable value, the Queue Master database review process calls the queue process task. The queue process task will query the database discovering the delayed jobs and retrieve them for processing. Again, this is only if the process handler is populated on the Queue Master.

###### Update Queue State

`NOTE:`Finally, at the completion of the review process the Queue Master will update the State Document to a state of reviewed. This is an important change in a distributed processing queue environment.

If the queue is currently quiet with no jobs being processed, there is nothing to prompt the Queue objects to go to work. Whilst time is passing some jobs in the queue may become available for processing due to their ___dateEnable___ value. As soon as the current date has past the jobs ___dateEnable___ date, the job is ready for processing.

To remedy this situation and to initiate processing of delayed jobs, the Queue Master review process completes by changing the State Document. This change is detected by all Queue objects connected to the same queue. If a Queue object detects a state update defined as reviewed, it will initiate a process restart function to query the database for more work.

___