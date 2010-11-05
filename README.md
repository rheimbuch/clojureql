ClojureQL
=========

ClojureQL is an abstraction layer sitting on top of standard low-level JDBC SQL integration.
It let's you interact with a database through a series of objects which work as Clojure data
type.

ClojureQL provides little to no assistance in creating specialized query strings, so that
compatability with the database backend is left to the user.

ClojureQL is modeled around the primitives defined in Relational Algebra.
http://en.wikipedia.org/wiki/Relational_algebra

For the user this means that all queries compose and are never executed unless dereferenced
or called with a function that has the ! suffix.

As a help for debugging, wrap your statements in (binding [*debug* true]) to see the
resulting SQL statement.

This project is still in the pre-alpha design phase, input is welcomed!

Initialization
--------------

    (def db
     {:classname   "com.mysql.jdbc.Driver"
      :subprotocol "mysql"
      :user        "cql"
      :password    "cql"
      :subname     "//localhost:3306/cql"})

Query
-----

    (def users (table db :users [:id :name]))  ; Points to 2 colums in table users

    @users
    >>> ({:id 1 :name "Lau"} {:id 2 :name "Christophe"} {:id 3 :name "Frank"})

    @(-> users
         (select (< {:id 3}))) ; Only selects IDs below 3
    >>> ({:name "Lau Jensen", :id 1} {:name "Christophe", :id 2})

    @(-> users
         (select (< {:id 3}))
         (project #{:title}))  ; <-- Includes a new column
    >>> ({:name "Lau Jensen", :id 1, :title "Dev"} {:name "Christophe", :id 2, :title "Design Guru"})

    @(-> users
         (select (!= {:id 3})))  <-- Matches where ID is NOT 3
    >>> ({:name "Lau Jensen", :id 1} {:name "Christophe", :id 2})

    @(-> users
         (select (both (= {:id 1}) (= {:title "'Dev'"}))))
    >>> ({:name "Lau Jensen", :id 1})

    @(-> users
         (select (either (= {:id 1}) (= {:title "'Design Guru'"}))))
    >>> ({:name "Lau Jensen", :id 1} {:name "Christophe", :id 2})

**Note:** No alteration of the query will trigger execution. Only dereferencing will!

Aggregates
----------

Coming soon!

Manipulation
------------

    @(conj! users {:name "Jack"})
    > ({:id 1 :name "Lau"} {:id 2 :name "Christophe"} {:id 3 :name "Frank"} {:id 4 :name "Jack"})

    @(disj! users {:name "Jack"})
    > ({:id 1 :name "Lau"} {:id 2 :name "Christophe"} {:id 3 :name "Frank"})

**Note:** These function execute and return a pointer to the table, so the can be chained with other calls.

Compound ops
------------

Since this is a true Relational Algebra implementation, everything composes!

    @(-> (conj! users {:name "Jack"})   ; Add a row
         (disj! (= {:name "Lau"}))      ; Remove another
         (sort :id :desc)               ; Prepare to sort in descending order
         (project #{:id :title})        ; Include these columns in the query
         (select (!= {:id 5}))          ; But filter out ID = 5
         (limit 10))                    ; Dont extract more than 10 hits
    > ({:id 3 :name "Frank"} {:id 2 :name "Christophe"})

**Note:** This executes SQL statements 3 times in this order: conj!, disj!, @

Joins
------

    (def visitors (table db :visitors [:id :guest]))

    @(join users visitors :id)                       ; USING(id)
    > ({:id 1 :name "Lau" :guest "false"} {:id 3 :name "Frank" :guest "true"})

    @(join users visitors #{:users.id visitors.id})  ; ON users.id = visitors.id
    > ({:id 1 :name "Lau" :guest "false"} {:id 3 :name "Frank" :guest "true"})

Helpers
-------

**(where) is not used in the relation model as the string notation is not yet available**

Below is just for inspiration. View the function (test-suite) in core.clj instead!

    ;;; (where "(%1 < %2) AND (avg(%1) < %3)" :income :cost :expenses)
    ;;; > "WHERE (income < cost) AND (avg(income) < expenses)"

    ;;; (where-not (either (= {:id 4}) (>= {:wage 200})))
    ;;; > "WHERE not ((id = 4) OR (wage >= 200))"

    (where (both (= {:id 4}) (< {:wage 100})))
    > "WHERE ((id = 4) AND (wage < 100))"

    (where (!= {:a 5})
    > "WHERE a != 5"

    (where (either (= {:id 2}) (> {:avg.id 4})))
    > "WHERE (id = 2 OR avg(id) > 4)"

    (-> (where "id > 2") (group-by :name))
    > "WHERE id > 2 GROUP BY name"

    (-> (where "id > 2") (order-by :name))
    > "WHERE id > 2 ORDER BY name"

    (-> (where "id > 2") (having "id=%1 OR id=%2" 3 5))
    > "WHERE id > 2 HAVING id=3 OR id=5"

Valid operators in predicates are: or  and  =  !=  <  <=  >  >=