Mongo db console commands

//showing the existing dbs..
show dbs
//use test
switching to db test, (only creating it when actually adding new data)
//prompts the name of the working db now
db
//the fllw would prompt the count(), in the link2 collection, in the current db...
>db.links2.count()
//inserting a record in links2
db.links2.insert({title:"unn titulo", url:"", comment:"", tags:["un primer tag", "un segundo tag"], saved_on: new Date})
//working with an object the javascript way...
data = {} | data.title = "un titulo" | data.tags = ["un tag", "otro"] | data.meta = {} | data.meta.OS = "win7" | db.links2.insert(data)
//printing the result of the find, in the structured json format.
db.links2.find().forEach(printjson)		//--> in this case we pass to forEach the printjson function...
//retriving only the first of the results of the find method.
db.links2.find()[0]		db.links2.find()[0]._id
//getting the timestamp present in the _id variable (is made of (also) the time it was created)
db.links2.find()[0]._id.getTimestamp()

/*the following function creates, when called, a new collection inside the same working db, that tracks the last id number we are in. This allows having the same behavieur than in relational DBs.*/
//apparently, u have to declare this function...
function counter(name) {
	var ret = db.counter.findAndModify({query:{_id:name}, update:{$inc:{next:1}},
	"new":true,
	upsert:true});
	return ret.next;
}
//so u can do something like
db.products.insert({_id:counter("products"), nombre:"primer nombre"})
//the result is something like:
{
   "_id": 1,
   "name": "un producto" 
}

{
   "_id": 2,
   "name": "otro producto" 
}
/*referencing in MongoDB*/
db.users.insert({name:"Richard"})
var a = db.users.findOne({name:"Richard"})
db.links2.insert({title:"primer titulo", author:a._id})		//reference to other collection throught the _id key...
//quering
db.users.findOne({ _id:link.author })		//a way to make manual inner joins... within the user db, we search for a coincidence of our _ids on the links2 db, author field.
---note--- 
embedding is much more efficient when we have significantly more read than writes. Otherwise, consider using the normalized way. These depends on every case.
/**/

#importing data from a .js in json format. With mongod running or in a services:
> ../../../mongodb/bin/mongo 127.0.0.1/bookmarks bookmarks.js
//the first part is the location to the mongo exe in the mongo usual location
//the second part is the server and db in which we will be importing in
//the third part is the file with all the mongo commands...
--this bookmarks file is in C:\Tuto\mongo\trying -- https://raw.github.com/tuts-premium/learning-mongodb/master/08%20-%20bookmarks.js

/*bookmarks.js extract*/
var u1 = db.users.findOne({ 'name.first': 'John' }),
    u2 = db.users.findOne({ 'name.first': 'Jane' }),
    u3 = db.users.findOne({ 'name.first': 'Bob' });

db.links.insert({
    title: 'Nettuts+',
    url: 'http://net.tutsplus.com',
    comment: 'Great site for web dev tutorials',
    tags: ['tutorials', 'dev', 'code'],
    favourites: 100,
    userId: u1._id
});
/**/
//connecting directly to db bookmarks
> ../../../mongodb/bin/mongo bookmarks

//searching in the collection all docs that have inside the tags array the "code" element.
//this can be done because we are dealing with an array --> array advantages...
db.users.find({tags:"code"}).forEach(printjson)

//with findOne u can do (not with find) findOne().name
db.links.find({favourites:100}, title:true, url:1)	//selecting only some fields...
db.links.find({favourites:100}, tags:0)	//selecting all but the tag field...

//selecting inside an object...
db.users.findOne({"name.first": "John"})
db.users.findOne({"name.first": "John"}, "name.last":1)

var john = db.users.findOne({"name.first": "John"})
db.links.find({userId:john._id}, {title:1, _id: 0})

/*queries directives*/
//greater than 150
db.links.find({favourites:{$gt:150}}, {_id:0, favourites:1, title:1}).forEach(printjson)
db.links.find({favourites:{$gt:150}}, {_id:0, favourites:1, title:1}).count()
//less than
db.links.find({favourites:{$lt:150}}, {_id:0, favourites:1, title:1}).forEach(printjson)
//$lte, $gte  -- and iqual
//using in
db.users.find({"name.first":{$in:["John", "Jane"]}})
//the opposite is $nin
db.users.find({"name.first":{$nin:["John", "Jane"]}})
//$all -- only the records with all the specifications in "tags" field.
db.links.find({tags: {$all:["code", "marketplace"]}}, {title:1, tags:1, _id:0})
//$ne -- not equal
//the $or flag search for the fullfillment of at least one of the elements in the array passed...
db.users.find({$or: [{"name.first": "John"}, {"name.last": "Wilson"}]})
//the opposite: $nor
//inclusive: $and
//$exists
db.users.find({email: {$exists: true}})
//$mod
db.links.find({favourites: {$mod: [5, 0]}}, {_id:0, title:1, favourites:1})
db.links.find({favourites: {$not: {$mod: [5, 0]}}}, {_id:0, title:1, favourites:1})
//elemMatch -- inside logins, search for an element match that has minutes = 20, and return the complete record
db.users.find({logins: {$elemMatch: {minutes: 20}}})
//searching for an 'at' prior to 2012/03/30.. and returning the whole record...
db.users.find({logins: {$elemMatch: {at: { $lt: new Date(2012, 3, 30)}}}})
//using where -- c) is equivalent to a)
a)	db.users.find({ $where: 'this.name.first === "John"'})
b)	db.users.find({ $where: 'this.name.first === "John"', age:30})
c)	db.users.find( 'this.name.first === "John"')
//injecting functions in mongodb -- as this example returns trueéfalse, its going to return values randomly
var frand = function() {return Math.random() > 0.5}
db.users.find(frand)
//
var f = function() { return this.name.first === "John"}
db.users.find(f)
//or
db.users.find( {$where: f} )

//other queries
//distinct -- returns a list of diff results
db.links.distinct('favourites')	--> [100, 32, 21, 78, ...]
db.links.distinct("url")


db.links.group({ 
	key:{userId : true}, 
	initial:{favCount: 0}, 
	reduce: function (doc, o) {o.favCount += doc.favourites}, 
	finalize: function(o) {o.name = db.users.findOne({ _id: o.userId}).name } }); ***
//the final part is not working...

db.links.group({ 
	key:{userId : true}, 
	initial:{favCount: 0}, 
	reduce: function (doc, o) {o.favCount += doc.favourites} });

db.links.group({ key:{userId : true},  initial:{favCount: 0},  reduce: function (doc,
o) {o.favCount += doc.favourites},  finalize: function(o) {o.name = "richard"}} );

//regex
db.links.find({ title: /tuts\+$/})
db.links.find({ title: {regex: /tuts\+$/}}, {title:1})

//counting
db.users.count({'name.first': 'John'})
db.users.count(); //all users in the collection
//sorting, limit
db.links.find({}, {title:1}).sort({title:1}).limit(1)	//1: asc -1: desc
//sorting, skipping and limiting... normal behavieur in the pagination rutine...
 db.links.find({}, {title:1, _id:0}).sort({title:1}).skip(3).limit(3)

/*updating*/ //by replacement or by modification...
---general form
/*
db.collection.update(
                      <query>,
                      <update>,
                      {
                        upsert: <Boolean>,	//if not found insert
                        multi: <Boolean>,	//change in all the condition <query> is fullfilled
                      }
                    )
*/
// more info in http://docs.mongodb.org/manual/reference/method/db.collection.update/
db.users.update({-the query object-}, {-the update object-}, -upsert boolean-);

var n = {title:"Nettuts+"}
db.links.find(n, {title:1})
db.links.update(n, {$inc: {favourites: 5}})

var q = {"name.last": "Doe"}
db.users.find(q, {name:1})
//we can use set to update a field or add a completly new one...
db.users.update(q, {$set: {"name.last": "Doetix"}})				//modifying an existing field..
db.users.update(q, {$set: {"email": "doetix81@gmail.com"}})		//inserting a new one...
//to remove a field w use unset
db.users.update(q, {$unset: {job: "Web developper"}})

db.users.update({"name.first":"John"}, {$set: {job:"Web developer"}}, false, true)
//modifying and then inserting an object
var bob = db.users.findOne({"name.first":"Bob"})
>bob
{
        "_id" : ObjectId("525f06242df9763abe646b62"),
        "name" : {
                "first" : "Bob",
                "last" : "Smith"
        },
        "age" : 31,
        "email" : "bob.smith@gmail.com",
        "passwordHash" : "last_password_hash"
}
> bob.job = "Thick Brush Painter"
> db.users.save(bob)
//find and modify -- findAndModify {{}}
/*
The findAndModify command atomically modifies and returns a single document. By default, the returned document does not include the modifications made on the update. To return the document with the modifications made on the update, use the new option.
{
  findAndModify: <string>,
  query: <document>,
  sort: <document>,
  remove: <boolean>,	//one of   |
  update: <document>,	//this two |
  new: <boolean>,		//if the new object must be shown or the old one..
  fields: <document>,	//fields to show in the result
  upsert: <boolean>
}
*/
> db.links.findAndModify({ 
	query:{favourites: {$gt:150}}, 
	sort:{title:1}, 
	update:{favourites: 333}, 
	new: true, 
	fields: {_id:0} });


//pulling into arrays
db.links.update(n, { $push: {tags: "jobs"}})
> db.links.findOne(n).tags
//several...
db.links.update(n, {$pushAll:{tags: ['blogs','press','contests']}})
//on pull into the array if the new element is not present..
db.links.update(n, {$addToSet:{tags: "dev"}})
//doing the same with an array...
db.links.update(n, {$addToSet:{ tags:{$each:  ["dev", "interviews"]} }})
//pulling out content from the array...
db.links.update(n, {$pull: {tags:'interviews'}})
//pulling several...
db.links.update(n, {$pullAll: {tags: ['blogs','dev', 'contests']}})
//poping out from the beginning or the end..
db.links.update(n, {$pop: {tags: 1}})   //--from the end (-1 -- from the beginning)
//positional operator... only the subobject gets updated...
db.users.update({'logins.minutes': 20} , {$inc:{ 'logins.$.minutes': 10}}, false, true)
db.users.update({'logins.minutes': 20} , {$set:{ 'logins.$.location': 10}}, false, true)

db.users.update({'logins.minutes': 30}, {$set: {random: true}}, false, true)

//renaming the fields name...
db.links.update({url: {$exists: true}}, {$rename:{"url": "camino"}}, false, true);
//more info on the positional operator in: http://docs.mongodb.org/manual/reference/operator/update/positional/
//taken from there:
/*
The positional $ operator facilitates updates to arrays that contain embedded documents. Use the positional $ operator to access the fields in the embedded documents with the dot notation on the $ operator.
db.collection.update( { <query selector> }, { <update operator>: { "array.$.field" : value } } )
*/
/***EXAMPLE
Consider the following document in the students collection whose grades field value is an array of embedded documents:

{ "_id" : 4, "grades" : [ { grade: 80, mean: 75, std: 8 },
                          { grade: 85, mean: 90, std: 5 },
                          { grade: 90, mean: 85, std: 3 } ] }
Use the positional $ operator to update the value of the std field in the embedded document with the grade of 85:

db.students.update( { _id: 4, "grades.grade": 85 }, { $set: { "grades.$.std" : 6 } } )
***/

//removing
db.users.remove({'name.first': "John"})
//all the collections in the selected db...
show collections
//dropping completly a collection...
db.acoll.drop()

//indexes...
db.links.find().explain
db.links.ensureIndex({ title: 1}) //in ascending order.. in mainly important in cpompund indexes..
//a reflect of this index can be found in that db indexes collection 
db.system.indexes.find();
//u cound put an index to a canging value, but every time u change that value the index must be updated. keep in mind.
//usually is a good idea to set the indexes at the beginning when no data is present in the collections. However, u could use the following formula to treat duplicates and unique data
//keeping only the first one, deleting the others..
db.links.ensureIndex({ title: 1}, { unique: true, dropDups: true})
//when considering the case of some of the documents without the idexed field, to save mongo from storing space for this index if the field itself has not been inserted:
db.links.ensureIndex({ title: 1}, {sparse: true})
//its important to think of the compund index as a nested one, an index of an index. Its related to each problem-case. Like in the case of the recepies: indexing first the ingredient and the the recepie, makes more sense than indexing in reverse. Its all related on how u are going to search.

db.links.ensureIndex({ title: 1, url: 1})   //this one means that u can search on title; or on title and url...
db.links.ensureIndex({ a: 1, b: 1, c: 1})   //searches are possible on a; a, b; a, b, c
//deleting indexes
db.links.dropIndex("title_1"); //the same way that appears in system.index collection...

/*concepts to follow*/
//Sharding and Replica Set...
http://www.slideshare.net/Dataversity/common-mongodb-use-cases-13695677
http://docs.mongodb.org/ecosystem/use-cases/product-catalog/

db.collection.update({"grades.grade":80}, { $set: {"grades.$.std": 18}})
