---
layout: post
title: Migrating Production Data with MongoDB 
---

When it comes to migrating production data with MongoDB it is extremely important to take care given MongoDB’s schemaless design which allows for little to no validation of incorrect data being inserted into your database. 

To ensure high SLA and prevent any mistakes from causing downtime with your customers, it is important to first create a staging or test database that you can test your migration script on before moving to production. In this blog post we will discuss important things to keep in mind during each step of your migration, from testing, to staging to deployment and production and how to rollback when things go wrong

## Creating a Test Environment

**Note**: This guide assumes you have mongo and docker installed in your environment

### 1. Get a snapshot of your production database with mongodump
```
mongodump --out /opt/backups/mongodb --host url_of_your_db --port port_of_your_db  --db your_db  
```

### 2. Create Docker mongodb container 
```
docker run --volume=/opt/backups/:/mongodb_backups/ --name migrationTest -d mongo:latest
```

### 3. Restore data into local mongodb database 
```
docker exec -it migrationTest bash -c “mongorestore /mongodb_backups/mongodb_dump”
```

## Create your migration script
For purposes of this tutorial, we will be assuming our data includes one collection called “pets”, and inside that collection we want to change the breed property to “labradoodle” if the pet is over the age of 5. We will be performing a migration that strips the urls of their protocol (if it the url has it).

For purposes of this example we will be using `migrate-mongo`, a npm package built for mongo migrations, but you can do this with any language/tool as long as it has a native driver to connect to MongoDB.

### 1. Install all required dependencies
```
npm install -g migrate-mongo
npm install mongodb
```

### 2. Initialize Migrate-Mongo and create your project folder
```
mkdir migration_test
migrate-mongo init
```

### 3. Configure migrate-mongo 
In the migrate-mongo-config.js file, change the below properties to ensure it will work with your test database
```
module.exports = {
  mongodb: {
    url: "mongodb://migrationTest", //CHANGE THIS
 
    databaseName: “testdb”, //CHANGE THIS
 
    options: {
      useNewUrlParser: true // removes a deprecation warning when connecting
      //   connectTimeoutMS: 3600000, // increase connection timeout to 1 hour
      //   socketTimeoutMS: 3600000, // increase socket timeout to 1 hour
    }
  },
 
  // The migrations dir, can be an relative or absolute path. Only edit this when really necessary.
  migrationsDir: "migrations",
 
  // The mongodb collection where the applied changes are stored. Only edit this when really necessary.
  changelogCollectionName: "changelog"
};
```

### 4. Initialize a new migration script
```
$ migrate-mongo create remove_labradoodles
```

### 5. Write your migration script
**Note**: It is important to add data to ensure that you can rollback your migration in case it causes any unexpected problems during production rollout.
```
module.exports = {
  up(db) {
    return db.collection(‘pets’).update({ age: { $gt: 5 }, breed: “labradoodle” }, { $set: { breed: “golden_retriever”, breed_migrated: true } }, { multi: true });
  },
 
  down(db) {
    return db.collection(‘pets’).update({ age: { $gt: 5 }, breed_migrated: true, breed: “golden_retriever” }, { $set: { breed: “labradoodle” }, $unset: { breed_migrated: “” } }, { multi: true });
  }
};
```

### 6. Create migration test
Before running our migration, we want to make sure we can validate that it works as expected. To do so, we will create a short script to query the database and make sure that the data has been modified as we intended.

```validate_migration.js
const MongoClient = require('mongodb').MongoClient;
const assert = require('assert');
 
// Connection URL
const url = 'mongodb://migrationTest'; //CHANGE THIS
 
// Database Name
const dbName = 'dbtest'; //CHANGE THIS
 
// Function that validates our migration
const validatePetMigration = function(db, callback) {
  // Get the pets collection
  const collection = db.collection(‘pets’);
  
  collection.find({
  	age: {
  		$gt: 5
  	},
  	breed: “labradoodle”
  }, function(err, pets){
  	assert.equal(err, null);
  	assert.equal(0, pets.length);
  	callback();
  })
}
 
 
// Use connect method to connect to the server
MongoClient.connect(url, function(err, client) {
  assert.equal(null, err);
  console.log("Connected successfully to server");
 
  const db = client.db(dbName);
  
  validatePetMigration(function(){
    client.close();
  });
 
});
```

### 5. Run your migration script
```  migrate-mongo up ```

### 6. Validate your migration
```  node validate_migration.js ```
Assuming nothing has gone wrong and assert does not throw any errors our migration was successful!

However if you do happen to run into errors, you can run ` migrate-mongo down ` to rollback the migration.

## Deploying our Migration 
Before we start deploying our migration script, we first need to make it our validation script runnable from JS.
```testmigration_deployment.js

const MongoClient = require('mongodb').MongoClient;
 
 // Function that validates our migration
const validatePetMigration = async function(db) {
	// Get the pets collection
	const collection = db.collection(‘pets’);
	
	try {
		var pets = db.collection(‘pets’).find({
	  		age: {
	  			$gt: 5
	  		},
	  		breed: “labradoodle”
  		});
  		
  		const migrationIsValid = (0 === pets.length);
  		return Promise.resolve(migrationIsValid);
	} catch(err) {
		return Promise.reject(err);
	}
}
 
module.exports = {
	isPetMigrationValid: async function(dbUrl) {
		try {
			const db = await MongoClient.connect(dbUrl);
			const isValid = await validatePetMigration(db)
			db.close();
			return isValid;
		} catch(err) {
			return Promise.reject(err);
		}
	}
}
```

We then need to automate our migration steps into a script by using with migrate-mongo’s API.
```run_migration.js
const {
  init,
  create,
  database,
  config,
  up,
  down,
  status 
} = require('migrate-mongo');
const MigrationChecker = require(“./validate-migration”);
const MongoClient = require('mongodb').MongoClient;

// Connection URL
const mongoUrl = 'mongodb://migrationTest/dbtest'; //CHANGE THIS

try {
	const db = await database.connect();
	const migrated = await up(db);
	const is_valid_migration = await MigrationChecker.isPetMigrationValid(mongoUrl);
	
	if (!is_valid_migration) {
		const rolled_back = await down(db);
		console.warning(“Migration was not successful”); 
	} else {
		console.log(“Migration was successful!”);
	}
	db.close();
} catch (err) {
	db.close();
	throw err;
}
```


After you are happy with your migration on your test instance the next step is run it on your staging server. When running it on your staging server, make sure you that your stage server has the same database, codebase and environment as production to catch as many possible production errors as possible. We recommend running your test suite on your staging server as well as manual testing of your product’s UI to catch any errors before moving to production.

When you are confident with your migration you are now ready to deploy it to production. Here at GeneralAvailability we use continuous integration with git for easy deployment, so we deployed our migration to production by committing our changes and pushing to master. We then ssh’d into our production server and ran our migration script along with our validation script to make sure everything was working.

And that’s it! You’ve just completed your first production migration with MongoDB. If you have any questions about this tutorial feel free to reach out to me at arielle@generalavailability.co.





