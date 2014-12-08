How Queries Were Added to Fortune Endpoints
===========================================

MAK: 12/7/2014

STOCK FORTUNE CODE
------------------

The first place that needs to change was "routes.js". Here's the relevant GET endpoint (line 264 in the Fortune original):

        /*
        * Get a list of resources.
        */
      router.get(collectionRoute, function (req, res) {
        var ids = [];
    
        if (typeof req.query.ids === 'string') ids = req.query.ids.split(',');
        if (typeof req.query.ids === 'object') ids = req.query.ids;
    
        // get resources by IDs
        adapter.findMany(model, ids)
    
        // do after transforms
        .then(function (resources) {
          return RSVP.all(resources.map(function (resource) {
            return afterTransform(resource, req, res);
          }));
        }, function (error) {
          sendError(req, res, 500, error);
        })
    
        // send the response
        .then(function (resources) {
          var body = {};
    
          body[collection] = resources;
          sendResponse(req, res, 200, body);
        }, function (error) {
          sendError(req, res, 500, error);
        });
    
      });

This code only looks for "ids" in the HTTP query. In the form:

    http://host/guests?ids=12345,6789
    
Everything else is ignored. So, this is the first place we need to extend to do anything more sophisticated. 

The next thing to look at is the adapter.findMany() call. This calls the following method in "adapters/nedb.js":

    adapter.findMany = function (model, query, limit) {
      var _this = this;
      if (_.isArray(query)) {
        query = query.length ? {_id: {$in: query}} : {};
      } else if (!query) {
        query = {};
      } else if (typeof query == 'number') {
        limit = query;
      }
      model = typeof model === 'string' ? this.model(model) : model;
      limit = limit || 1000;
    
      return new RSVP.Promise(function (resolve, reject) {
        model.find(query, function (error, resources) {
          if (error) return reject(error);
          resources = resources.map(function (resource) {
            return _this._deserialize(model, resource);
          });
          resolve(resources);
        });
      });
    };
    
Note that this is called from the get method above WITHOUT a limit set. That means stock Fortune is limited to 1000 results. 
Not cool! Or that _would_ be the case if the `limit` variable was actually used in this method, but it isn't (there is no NeDB 
`limit()` call. It seems like the author intended to support limits, but never got around to it.

The `findMany` method calls the actual NeDB `find` method with `model.find()` (inside the returned Promise). Finally, we are down 
to the bare metal.

OK, so out of the box, the only thing supported is a list of ids passed as a get parameter. To enable access
to the full range of NeDB `find`, `sort` and `limit` we need to pass the objects that control these to the `model.find` through the 
http query string itself.

The most powerful way would be to pass the actual JSON object as a parameter like:

    http://localhost/guests?$_find={"firstName":"Mitch"}&$_sort={"lastName":-1}
    
But we would probably need to uuencode the JSON since this is chock full of special characters like `{, $` and :`. But for emergency
situations, we could add this capability as a hidden feature. It's not in this version (anymore).

It would ne nice to be able to just pass field names like:

    http://localhost/guests?firstName="Mitch"&lastName="Kahn"
    
But this will requiring a bit of `hasOwnProperty` checking- not a huge deal but also not in this version. In fact,
you could blow off the hasOwnProperty and just let the find() fail to find anything if a non-exsitant property
is passed. I think this would be a nice convenience capability to add later.
    
For this version, I went with find[], sort[] and limit[] methods like so:

    http://localhost:8503/guests?find[firstName]=Mitch&find[lastName]=Kahn
    
and that results in a `req.query.find = { firstName: "Mitch", lastName: "Kahn" }`.  That's great for simple
equivalence queries, but what about stuff like: `{ satellites: { $lt: 'Amos' }`? We could just put the comparitor
in the string like so:

    http://localhost/guests?find[lastName]="$lt:Kahn"
    
and define the common ones: $lt, $gt, $lte, $gte. In this version, all the NeDB operators except $regex are supported.


Descending Into Objects
-----------------------

We can add objects as properties on Fortune APIs. To find or sort on sub properties, we can use a syntax like:

    http://localhost/guests?find[boswell:answers:q1]="$gt:5"
    
But this will be in a future rev. `{ bosell: { answers: { q1: { $gt: 5} } }`


API EXTENSION
=============

FIND
----

The find function has the following calls:

*   EQUIVALENCE: `find[<model-property>]="<value>"` any number can be strung together (they are ANDed). ORing is not supported.

Example:  

    GET guests?find[firstName]=Mitch&find[lastName]=Kahn
    
GETs all Guests with firstName=Mitch AND lastName=Kahn.

*   RELATIVE: `find[<model-property>]='$<comparitor>:<value>` and number can be combined, they are ANDed.

Example:

    GET guests?find[created]="$gt:14234567"
    
Gets all guests created after a given time. All comparitors are supported including the array operations. For example, to
look for several first names:

    GET guests?find[firstName]=$in:Mitch,John,Scott,Billy


SORT
----

The sort function has the following call:

*  `sort[lastName]="<1/-1>"`  1= ascending, -1=descending. Everything else is ignored.

Example:

    GET guests?find[created]=$gt:12345&sort[lastName]=-1
    
Gets all guests created after a certain time, sorted by lastName, Z to A.

Support for multiple sort keys is supported and they are processed in the order in the url:

    GET guests?sort[lastName]=1&sort[firstName=1]

LIMIT
-----

The limit function is very simple:

*  `limit="<number>"` Default is no limit.

Example:

    GET guests?find[created]="$gt:12345"&sort[lastName]=-1&limit=50
    
There is a hook for `limit` in stock Fortune, but it is not used.

IDs
---

The original ids=[comma sep list] is still supported for backwards compatibility.


SPECIFIC CHANGES to STOCK FORTUNE
=================================

fortune.js
----------

A comment at line 84 to track _init. No other changes.

route.js
--------

Removed functionality that parsed the query string and pushed that down into the adapter. You could debate where
the line should go between the database and the http query, but the Fortune had it was inflexible. This way, an
adapter writer knows what the inbound query string should look like and can directly translte it into the native
DB query. No middle step involved. All the work here was commenting out the id parser and passing the query directly
to a new adapter method: `findManyFiltered`. Everything after that call is stock Fortune.

    router.get(collectionRoute, function (req, res) {
        var ids = [];
    
        //MAK leaving this in for backwards compatibility
        //But the code is moved to the adapter where it really belongs when generalized
        //if (typeof req.query.ids === 'string') ids = req.query.ids.split(',');
        //if (typeof req.query.ids === 'object') ids = req.query.ids;
    
        // MAK just passing the whole query object, let the adapter do the right thing
        adapter.findManyFiltered(model, req.query)
    
        // do after transforms
        .then(function (resources) {
          return RSVP.all(resources.map(function (resource) {
            return afterTransform(resource, req, res);
          }));
        }, function (error) {
          sendError(req, res, 500, error);
        })
    
        // send the response
        .then(function (resources) {
          var body = {};
    
          body[collection] = resources;
          sendResponse(req, res, 200, body);
        }, function (error) {
          sendError(req, res, 500, error);
        });
    
      });


nedb.js (Database Adapter)
--------------------------

Added the following `findManyFiltered` method:

    /**
     * Replacement method for the limited findMany
     * @param model
     * @param query the query string right out of the URL
     * @returns {rsvp$umd$$RSVP.Promise}
     */
    adapter.findManyFiltered = function (model, query) {
        var _this = this;
    
        var arrayOperators = [ '$in', '$nin' ];
        var unaryOperators = ['$gt', '$gte', '$lt', '$lte', '$ne', '$exists'];
        var supportedOperators = _.union(arrayOperators, unaryOperators);
    
        //Process find queries
        var findQuery = {};
        if (query.find){
            var keys = Object.keys(query.find);
            keys.forEach(function(key){
                var findTerm = query.find[key];
                // check for operator
                var splitFindTerms = findTerm.split(":");
                var qlen = splitFindTerms.length;
    
                if ( qlen === 1 ){
                    //This is a simple equivalence query (name=John)
                    findQuery[key] = query.find[key];
                }
                else if ( qlen === 2 ){
                    //This is either >,<,! query or a descender
                    if (_.contains(supportedOperators, splitFindTerms[0])) {
    
                        console.log("The find operator is: " + splitFindTerms[0]);
    
                        if (_.contains(unaryOperators, splitFindTerms[0])) {
                            //Unary operation
                            var searchObj = {};
                            // build { $gt: 2345 }, for example
                            searchObj[splitFindTerms[0]] = splitFindTerms[1];
                            findQuery[key] = searchObj;
    
                        } else {
                            //Array operation.
                            var searchObj = {};
                            var searchArray = splitFindTerms[1].split(",");
                            // build { $in: [x,y,z] }, for example
                            searchObj[splitFindTerms[0]] = searchArray;
                            findQuery[key] = searchObj;
    
                        }
                    } else {
                        //DESCENDER without operator, not supported
                        console.warn("Descending into objects without operator not supported yet.");
                    }
    
    
                } else  if ( qlen > 2 ) {
                    //Descender equivalence or relative, either way no in this version
                    console.warn("Descending into objects in any way not supported yet.");
                }
                else {
                    // This is a malformed query. Log and ignore.
                    console.warn("Malformed find query");
                };
            }); // end of forEach on find key
        } // end of find processing
    
        // format sort[field]=1/-1
        var sortQuery = {};
        if (query.sort){
    
            console.log("Sort query is: "+query.sort);
            var keys = Object.keys(query.sort);
            keys.forEach(function(key){
    
               if ( (query.sort[key]==1) || (query.sort[key]==-1) ){
                //valid choice
                    sortQuery[key] = query.sort[key];
               }
    
            });
        }
    
        var limit = null; // no limit
    
        if (query.limit){
            console.log("Limit query is: "+ query.limit);
            //Asking for a limit, validate
            var qLimit = parseInt(query.limit);
            if (!isNaN(qLimit)){
                limit = qLimit;
            }
        }
    
        // At this point we have the find, sort and limit fields set up for NeDB
        // The last step is backwards compatibility for the ids= query
    
        //TODO do we really need this? JSON spec thing??
    
        var idQuery = {};
        if (query.ids){
            var idQ = query.ids.split(',');
            if (_.isArray(idQ)) {
                idQuery = idQ.length ? {_id: {$in: idQ}} : {};
            }
        }
    
        // merge id query with find query
        findQuery = _.assign(findQuery, idQuery);
    
        model = typeof model === 'string' ? this.model(model) : model;
    
    
    
      return new RSVP.Promise(function (resolve, reject) {
        model.find(findQuery).sort(sortQuery).limit(limit).exec( function (error, resources) {
          if (error) return reject(error);
          resources = resources.map(function (resource) {
            return _this._deserialize(model, resource);
          });
          resolve(resources);
        });
      });
    };


settings.js (in SSEXY)
----------------------

Added a field "fortuneClassic". If set to `true` this uses the original module (no filters).

index.js (in SSEXY)
-------------------

Added an if/then/else to load stock fortune or fortunePlus based on the settings.


