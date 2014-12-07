How Queries Were Added to Fortune Endpoints
===========================================

MAK: 12/7/2014

The first place that needed to change was "routes.js". Here's the relevant GET endpoint (line 264 in the Fortune original):

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
We'll come back to this in a bit. 

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
Not cool! Or that _would_ be the case if the `limit` variable was actually used in this method, but it isn't. It seems like 
the author intended to support limits, but never got around to it.

The `findMany` method calls the actual NeDB `find` method with `model.find()` (inside the returned Promise). Finally, we are down 
to the bare metal.

OK, so out of the box, the only thing supported is a single id or a list of ids passed as a get parameter. To enable access
to the full range of NeDB `find` and `sort` we need to pass the objects that control these to the `model.find` through the 
http query string itself.

The most powerful way would be to pass the actual JSON object as a parameter like:

    http://localhost/guests?$_find={"firstName":"Mitch"}&$_sort={"lastName":-1}
    
We would probably need to uuencode the JSON since this is chock full of special characters like `{, $` and `:`. But for emergency
situations, I would leave this capability as a hidden feature.

It would ne nice to be able to just pass field names like:

    http://localhost/guests?firstName="Mitch"&lastName="Kahn"
    
But this will requiring a bit of `hasOwnProperty` checking- not a huge deal. This can be extended for common
query options like greater than and less than:

    http://localhost/guests?created[gt]=1000&created[lt]1500
    
Would search for guests created between 1000 and 1500. This exact url results in a `req.query` entry of:

    { created: { gt: 1000, lt: 1500 }}
    
very handy for synthesizing an NeDB query! Actually, we can re-cast the uuencode object $find method above as:

    http://localhost:8503/guests?find[firstName]=Mitch&find[lastName]=Kahn
    
and that results in a `req.query.find = { firstName: "Mitch", lastName: "Kahn" }`.  That's great for simple
equivalence queries, but what about stuff like: `{ satellites: { $lt: 'Amos' }`? We could just put the comparitor
in the string like so:

    http://localhost/guests?find[lastName]="$lt:Kahn"
    
and define the common ones: $lt, $gt, $lte, $gte. 

The find[] syntax effectively replaces the direct "lastName=Kahn" syntax above, so there's _no point in doing that one at all_.
And the find[] style syntax will wprk fine for sort as well. 

API EXTENSION
=============

FIND
----

The find function has the following calls:

*   EQUIVALENCE: `find[<model-property>]="<value>"` any number can be strung together (they are ANDed). ORing is not supported (yet)

Example:  

    GET guests?find[firstName]=Mitch&find[lastName]=Kahn
    
GETs all Guests with firstName=Mitch AND lastName=Kahn.

*   RELATIVE: `find[<model-property>]='$<comparitor>:<value>` and number can be combined, they are ANDed.

Example:

    GET guests?find[created]="$gt:14234567"
    
Gets all guests created after a given time. Supported comparitors are "$gt", "$gte", "$lt", "$lte".

SORT
----

The sort function has the following call:

*  `sort[lastName]="<1/-1>"`  1= ascending, -1=descending. Everything else is ignored (or 406?).

Example:

    GET guests?find[created]="$gt:12345"&sort[lastName]=-1
    
Gets all guests created after a certain time, sorted by lastName, Z to A.

Support for multiple sort keys is not in this version and may be overkill.
        


