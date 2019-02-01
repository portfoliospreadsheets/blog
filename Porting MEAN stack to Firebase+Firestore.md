# How to: Port a MEAN Stack application to Firebase+Firestore 

23rd January 2019. Updated: 1st February 2019.

## Executive Summary

I ported our web application [Portfolio Spreadsheets](https://www.portfoliospreadsheets.com/?ref=gh_porting) from using a MEAN stack to using a Firebase + Firestore stack. In the process I eliminated many paid service dependencies and some code dependencies. Below I provide code snippets and commentary for some common porting tasks. I provide a simple implementation for a given feature and link to the most relevant documentation to help you cut through the piles of information out there.

## Definitions

MEAN stands for MongoDB Express.js Angular and Node.js.

"Firebase is Google's mobile platform that helps you quickly develop high-quality apps and grow your business" but their stack can be used to develop web apps. Firebase includes the Realtime Database and the Firestore Database, as well as Firebase Hosting for hosting static sites and Single Page Applications (SPA).

## Serverless?

Firestore has client libraries that allow SPAs to manipulate the database and a set of security rules that can be applied to these manipulations. However, these rules are insufficient for serious data validation or more complex interactions or functions that can not be trusted to the client. For these cases we can use Firebase Cloud Functions. These functions run on Google's servers but no specific server is provisioned for your server-side code.

## Cloud Functions

### Execution notes
Cloud Functions support Python, Node.js 6, Node.js 8 (in beta), and Go execution environments. I will only talk about Node.js 8. For Node.js you can use Typescript but I will only discuss Javascript.

While Google may re-use the execution environment for faster startup, each function call runs in its own independent Node.js environment. You can store state in global scope but it is not guaranteed that subsequent calls to the function will use the same copy. If the function has not been called recently it may result in a cold start which results in the function taking several seconds to start up.

Google can start multiple function instances to scale up according to demand.

See <https://cloud.google.com/functions/docs/concepts/exec> for more details.

### Writing a Cloud Function

You have two options to write a cloud function - using an Express app, or the newer method, a callable function. If you want a traditional HTTP REST interface use the former. If you don't need REST, use the latter for convenience. See why here <https://stackoverflow.com/questions/49475667/is-the-new-firebase-cloud-functions-https-oncall-trigger-better/49476648#49476648>


#### Option 1: Express

```javascript
const express = require('express');
const cors = require('cors')({origin: true});

const app = express();
app.use(express.json());
app.use(cors);

app.post('*', async (req, res) => {
  // ... do usual Node.js / Express.js stuff
  return res.json({data: 'foo'});
});

//Expose the cloud function in functions/index.js of the Firebase project
const functions = require('firebase-functions');
exports.myFunction = functions.https.onRequest(app);
```

This will expose a HTTP endpoint on the server on a url like 'https://us-central1-my-app.cloudfunctions.net/myFunction/'. Do use a trailing / when calling the endpoint. You may use Express Routers to handle more complex routes.

See <https://firebase.google.com/docs/functions/http-events>

However, the code above does not perform authentication. For the server side see <https://github.com/firebase/functions-samples/blob/master/authorized-https-endpoint/functions/index.js> and the end of this article for the client side <https://blog.realworldfullstack.io/real-world-app-part-12-cloud-functions-for-firebase-8359787e26f3>.

#### Option 2: Callable Functions

Callable Functions handle most of the authentication boilerplate for you both on the server side and the client side. 

```javascript
const functions = require('firebase-functions');
exports.myFunction = functions.https.onCall((data, context) => {
  //data is the data passed to the call by the client

  //authentication is done this way
  if (!context.auth) return {status: 'error', code: 401, title: 'Authentication error',  message: 'Not signed in'};

  //see the next section on Custom Claims to implement additional claims such as isMember on the JWT token
  if (!context.auth.token.isMember) return {status: 'error', code: 403, message: 'You need more permissions to use this feature.'}; 

  //context has useful info such as:
  const userId = context.auth.uid; //the uid of the authenticated user
  const email = context.auth.token.email //email of the authenticated user

  //must return a value or promise from the function or it will not finish and will killed after a default timeout of 60 seconds
  return 0;
});
```

See <https://firebase.google.com/docs/functions/callable>

### Custom Claims

To manage authorization you can add custom claims to the authorization token. In the cloud function:

```javascript
const admin = require('firebase-admin');
await admin.auth().setCustomUserClaims(uid, {isMember: true});
```

There is additional documentation for using custom claims in the database rules, and reading the claim on the client here <https://firebase.google.com/docs/auth/admin/custom-claims>. If you change the claim on the server you can ask the client to reauthenticate:

```typescript
async reAuthenticate() {
  await this.afAuth.auth.currentUser.getIdToken(true);
}
```

### Calling a cloud function on the Angular 7 client

Call an cloud function written with Option 1 (Express) just like any other HTTP REST endpoint.

To call an callable function, we use AngularFire (npm i @angular/fire --save) instead of the Firebase SDK.

```typescript
import { Injectable } from '@angular/core';
import { AngularFireFunctions } from '@angular/fire/functions';

@Injectable()
export class MyService {

  constructor(private fns: AngularFireFunctions) {}

  callMyFunction() {
    const callable = this.fns.httpsCallable('myFunction');
    return this.handleError(callable({ data: 'foo' }));
  }

  handleError(listener: any) {
        return new Promise((resolve, reject) => {
            listener.subscribe((error) => {
                console.log('Server returned: ' + JSON.stringify(error));

                //define your own format for sending/returning data/errors
                if (error && error.status && error.status === 'ok') {
                    resolve(error);
                } else {
                    //add code to handle error and surface it to the user here
                    resolve({status: 'error'});
                }
        }, (err) => {
            resolve({status: 'error'});
        });
    });
  }
}
```

## Firebase Authentication

You can enable various sign-in providers such a Facebook and Twitter at https://console.firebase.google.com . Firebase handles the server-side authentication and session management. Therefore, you no longer need express-session. I used redis-connect for express-session and therefore removed a dependency on RedisLabs.com's database service. I was paying US$7 a month for their 100MB standard plan.

### Authentication on the Angular Client

I only used the Google sign-in provider. AngularFire manages all the token exchange and token storage on the client. So when you call a cloud function via AngularFire the JWT is automatically included. The following code is from the instructions here <https://github.com/angular/angularfire2/blob/master/docs/auth/getting-started.md>.

```typescript
import { Component } from '@angular/core';
import { AngularFireAuth } from '@angular/fire/auth';
import { auth } from 'firebase/app';

@Component({
  selector: 'app-root',
  template: `
    <div *ngIf="afAuth.user | async as user; else showLogin">
      <h1>Hello {{ user.displayName }}!</h1>
      <button (click)="logout()">Logout</button>
    </div>
    <ng-template #showLogin>
      <p>Please login.</p>
      <button (click)="login()">Login with Google</button>
    </ng-template>
  `,
})
export class AppComponent {
  constructor(public afAuth: AngularFireAuth) {
  }
  login() {
    this.afAuth.auth.signInWithPopup(new auth.GoogleAuthProvider());
  }
  logout() {
    this.afAuth.auth.signOut();
  }
}
```

However, the SDK and AngularFire do not return refresh tokens so if you need them from Google, you still have to write cloud functions and client code to handle authentication, then pass the credentials (tokens) to AngularFire to login to Firebase. In the cloud function save the refresh token, access token, expiry date, and email in the Firestore database. The email is the only way to match the tokens to the logged in user.

Pass the credentials from the cloud function back to the client and do the following to login using AngularFire:

```typescript
import * as firebase from 'firebase/app';

//component code omitted
private doCustomLogin(tokens) {
    return new Promise<any>((resolve, reject) => {
      const id_token = tokens.tokens.id_token;
      const access_token = tokens.tokens.access_token;

      const credentials = firebase.auth.GoogleAuthProvider.credential(id_token, access_token);

      this.afAuth.auth.signInAndRetrieveDataWithCredential(credentials)
      .then(res => {
        resolve(res);
      })
      .catch(err => {
        reject(new Error('signInAndRetrieveDataWithCredential error.'));
      });
    }
  );
}
```


## MongoDb vs. Firestore

### Price

mLab.com's lowest production ready database plan was US$15 per month for 1GB storage and unlimited read/writes. However they have been acquired by MongoDB.com who's product Atlas starts at about US$9 per month for 2GB plus $0.01+ per GB transferred. Firestore has the following pricing structure (note: prices for multi-regional database, regional prices lower https://cloud.google.com/firestore/pricing#pricing_update):

Resource | Cost | Free Quota
--- | --- | ---
Stored data | $0.18/GB | 1 GB total
Bandwidth |Google Cloud Pricing | 10GB/month
Document writes |$0.18/100K | 20K/day
Document reads |$0.06/100K | 50K/day
Document deletes |$0.02/100K | 20K/day

The free quota makes using Firestore essentially free for low usage scenarios. However, for some applications the document read/write/deletes quota may be easily exceeded and relatively expensive. The R/W/D charge is irrespective of the size of the document.

Although Firestore is still in beta and has no Service Level Agreement, the free Spark plan is using the same infrastructure as the paid plans. The paid plans include the free quota of the spark plan. So, it is recommended that you join the pay as you go Blaze plan to get all the features (such as the ability to call 3rd party network APIs from cloud functions) and set a daily spending limit.

### Similarities and Differences

Both MongoDB and Firestore are NoSQL databases that deal with documents rather than records.

In MongoDB you might store a user document like { _id: "server generated id", email: "test@example.com" } in the 'users' collection and do a find by _id or email. In Firestore you might store the document in the nested path 'users/\<uid>' where \<uid> is the id of the document which equals the user id. 'users' is the collection and 'uid' is the document within the collection. To save the document:

```javascript
var docRef = await db.collection('users').doc(context.auth.uid);
await docRef.set({email: 'test@example.com'});
```

For the callable cloud function, the authenticated user id is accessible via context.auth.uid. You could directly retrieve the document without a query like:

```javascript
const docRef = await db.collection('users').doc(context.auth.uid);
const docSnapshot = await docRef.get();
```

If your users could generate multiple reports, you could store it like 'reports/\<uid>/current/<report-id>'. The advantage is you wouldn't need to query reports where userId === uid, you could directly reference the collection of reports like:

```javascript
const querySnapshot = await db.collection('reports').doc(context.auth.uid).collection('current').get();
```

And access a specific report with id r1234567890 like:

```javascript
const docSnapshot = await db.collection('reports').doc(contex.auth.uid).collection('current').doc('r1234567890').get();
```

## Firestore Queries

In Firestore, query performance is proportional to the size of your result set, not your data set. In my port, I replaced Keen.io analytics with my own analytics event collection. Here is a query for the user ids of 'signups' events in the past day:

```javascript
const event = 'signups';
const yesterday = moment().subtract(1,'d').toDate();
let querySnapshot = await db.collection('analytics')
  .where('type', '==', event)
  .where('createdAt', '>', yesterday)
  .get();

var userIds = [];
if (!querySnapshot.empty) {
  docs = querySnapshot.docs;
  for(let i = 0; i < docs.length; i++) {
    let doc = docs[i];
    let uid = doc.get('userId');
    userIds.push(uid);
  }
}
```

You can use querySnapsot.docs.forEach((doc) => {}); but in Node.js 8 it runs in parallel and may have weird side effects. See <https://codeburst.io/javascript-async-await-with-foreach-b6ba62bbf404>

You can learn more about queries at <https://firebase.google.com/docs/firestore/query-data/queries>. Note especially the Query Limitations. Firestore queries are quite limited and you may have to do additional in-memory filtering, grouping, and sorting of the results yourself

### Realtime updates

On the Angular client side you can run realtime queries that get updated whenever the data in the collection changes. The advantage of using the user id in the path of collections is that you can easily write Firestore rules that restrict users to reading/writing their own data. Here is a section of the rules configuration file that only allows users to read/delete their own reports:

```javascript
service cloud.firestore {
  match /databases/{database}/documents {

	function authenticated() {
  	return request.auth != null;
  }
  
  function currentUser() {
    return request.auth;
  }
  
  match /reports/{userId}/current/{reportId} {
		allow read, delete: if authenticated() && (currentUser().uid == userId);	
  }
}
}
```

And here is the Angular typescript in a service that watches the reports collection and sends updates to subscribers:

```typescript
watchReports() {
  this.fbSubs.push(this.db
    .collection('reports/' + this.auth.getuid() + '/current')
    .snapshotChanges()
    .pipe(map(docArray => {
      return docArray.map(doc => {
        return {name: doc.payload.doc.data().name};
      });
    }))
    .subscribe((reports: any[]) => {
      this.reports = reports;
      this.reportsChanged.next([...this.reports]);
    }, error => {
      this.reportsChanged.next(null);
  }));
}

ngOnDestroy() {
  this.fbSubs.forEach(sub => sub.unsubscribe());
}
```

Previously I used the Ably.io realtime data network to communicate job status for jobs running on the job queue provided by the npm package Kue. Kue used a Redis database to manage jobs. Now with Firestore, cloud functions scale to the number of requests so a job queue is not necessary, and realtime updates let clients monitor the job status. From within the cloud function I can update the job status like:

```javascript
await docRef.update({syncStatus: 'Completed', syncMessage: 'Completed generating report.'});
```

### Getting around expensive read/writes/deletes

I previously mentioned Query Limitations, and the cost of read/writes and deletes to documents (e.g. $0.18 to write 100K documents). Because I have potentially thousands of small transactions to write and query, and later delete - I opted to write 1000 transactions in a batch to a single Firestore document. This potentially reduces the read/write/delete count by a thousand fold. I then can retrieve them to memory and query and filter them using my own code.

This is only about 10MB of data when the cloud function is allocated 256MB of memory and 400Mhz of processor by default. You can change this per cloud function, from 128MB/200Mhz to 2GB/2.4GHz. CPU and memory is cheap compared with database writes. One minute of 256MB/400Mhz costs $0.00028. It may seem odd to not use the database's querying capabilities but it makes sense financially and performance wise.

For similar reasons, querying the database to calculate totals over a collection of documents is expensive. See <https://hackernoon.com/how-we-spent-30k-usd-in-firebase-in-less-than-72-hours-307490bd24d?gi=9eda8387d0e9>. Aggregation queries are not supported (<https://firebase.google.com/docs/firestore/solutions/aggregation>) so you have to roll your own. To keep costs down, you should keep stats as you add/remove data.

For example, to keep count of the number of reports stored in the system, I write a cloud function trigger that fires whenever a report document is added. Ideally, this would be written as a transaction. See <https://firebase.google.com/docs/firestore/manage-data/transactions>.

```javascript
exports.onCreateReport = functions.firestore.document('reports/{userId}/current/{reportId}').onCreate(async (snap, context) => {
  try {
    const ref = await db.collection('stats').doc('general');
    let general = await ref.get();
    await ref.set({report_count: general.get('report_count') + 1}, {merge:true});

    return 0;
  } catch(err) {
    console.error('onCreateReport:' + String(err));
    return err;
  }
});
```
## Firebase Hosting

Firebase hosting is quite simple. You upload your static assets via the 'firebase deploy' CLI. It has 10GB of bandwidth per month free, and 1GB of storage free, SSL is free. Comparing with Heroku, to get SSL you need to pay $7/month for the Hobby plan. Heroku does have a theoretical allowance of 2TB of bandwidth, whereas with Firebase Hosting $7 would only get you 56GB.

Firebase Hosting has no HTTP logs available though you can see cloud function logs. With their global Content Delivery Network (CDN) I don't need to use CloudFlare anymore. However, CloudFlare has some DDOS (Distributed Denial of Service) attack protection that Firebase does not. Hence, it's important to set daily spending limits on your Firebase account.

I have not covered it here, but when you start your Firebase Hosting (Angular) project you initialise the directory with a firebase.json file by using the CLI tool ('npm install firebase-tools -g') command 'firebase init'. There you can select to link to a Firebase Hosting account, and you will be asked 'What do you want to use as your public directory? (public)'. Answer 'dist' for Angular. This will tell Firebase to upload the 'dist' directory to Firebase Hosting. Then build your Angular project as usual 'npm run build' and call 'firebase deploy'.

## Deployment

Because Firebase allows you to create multiple projects each with their own free quotas you can create a seperate project for development and production. I have not covered the basics of starting a Firebase project, but to do so you would have installed the CLI tools via 'npm install firebase-tools -g'.

To create a new environment run the command 'firebase use --add' and select your Firebase development or production project and give it an alias such as 'production'. Then to switch between projects run 'firebase use production' or 'firebase use development.

### Environment variables

To set an environment variable for the cloud functions use for example 'firebase functions:config:set stripe.key="abcdefgh"' to set the 'key' variable nested within the 'stripe' service. To view to environment variables do 'firebase functions:config:get' and you will get a JSON structure containing your variables like this:

```javascript
{
  "stripe": {
    "key":"abcdefgh",
  }
}
```

Access these variables in the cloud function like:

```javascript
const functions = require('firebase-functions');
const stripe_key = functions.config().stripe.key;
```

For more details see <https://firebase.google.com/docs/functions/config-env>.

### Deploy

'firebase deploy' deploys your Firebase code and database rules to the current Firebase project. To update cloud functions only, use 'firebase deploy --only functions'. To update a specific function or functions use 'firebase deploy --only functions:myFunction1,functions:myFunction2'.

### Debugging

Before deploying, firebase-tools runs eslint to check your javascript. In the Visual Studio Code editor I have eslint setup as well. This setup is unable to detect unknown references because it makes no assumptions about global variables. This is the most common error you'll see in your cloud functions logs from the Firebase console. You can use console.log() or console.error() in your code.

It is possible to debug cloud functions locally using an emulator but I have not tried it, see <https://firebase.google.com/docs/functions/local-emulator#use_the_cloud_functions_shell>


### Firestore Backups

Currently there is no automated backup service from Google for Firestore. You can create manual backups to a Cloud Storage bucket via these instuctions <https://firebase.google.com/docs/firestore/manage-data/export-import>.

### Refactoring cloud function code

In the Google code examples they put all functionality into functions/index.js. This becomes unmanageable. You can refactor the code like this:

admin.js
```javascript
const admin = require('firebase-admin');

admin.initializeApp();

var db = admin.firestore();
const settings = { timestampsInSnapshots: true};
db.settings(settings);

module.exports = {
  db, admin
}
```

index.js
```javascript
const functions = require('firebase-functions');

const {getReports, createReport, updateReport} = require('./reports');

module.exports = {
  'getReports': functions.https.onCall(getReports),
  'createReport': functions.https.onCall(createReport),
  'updateReport': functions.https.onCall(updateReport),
};
```

reports.js
```javascript
const {db} = require('./admin');

exports.getReports = (data, context) => {
  //
}

exports.createReport = (data, context) => {
  //
}

exports.updateReport = (data, context) => {
  //
}
```

## Conclusion and Summary

Firebase development is fast, convenient, and potentially cost saving if you make use of the Google services they bundle together. 

### Pros

In porting my MEAN stack application to Firebase + Firestore I managed to eliminate the following dependencies:
* mLab.com DbaaS US$15/month
* RedisLabs.com DbaaS US$7/month
* Heroku.com PaaS US$7/month
* Keen.io analytics ($1/10K events streamed, $1/10M properties scanned)
* Ably.io realtime data network (was on the free plan but paid plans start from $19.99/month)
* Amazon S3 storage was used for Ably.io backups (was under the free quota)
* CloudFlare CDN
* Express.js is only used for 3rd party integration REST endpoints
* express-session used the Redis database
* Kue job queue management used the Redis database
* authentication boilerplate code
* Firestore has guaranteed uptime — 99.999% for multi-region instances of Cloud Firestore, and 99.99% for regional instances

### Cons

* Debugging is slower with a deploy to debug workflow
* Database queries are not as rich as with MongoDB
* Database backups are not automated
* Cloud function cold starts
* SPA bundle size increased from 2MB to 2.5MB
* Weaker DDOS attack protection
* Dependence on Google and closed-source code
