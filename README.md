codeigniter-base-model
=====================================

This is an update to CodeIgniter Base Model, original by [jamierumbelow](https://github.com/jamierumbelow/codeigniter-base-model).


Installation/Usage
------------------

Download and drag the MY\_Model.php file into your _application/core_ folder.

Extend your model classes from `MY_Model` and all the functionality will be baked in automatically.


Callbacks/Observers
-------------------

There are many times when you'll need to alter your model data before it's inserted or returned. This could be adding timestamps, pulling in relationships or deleting dependent rows. The MVC pattern states that these sorts of operations need to go in the model. In order to facilitate this, **MY_Model** contains a series of callbacks/observers -- methods that will be called at certain points.

The full list of observers are as follows:

* $before_create
* $after_create
* $before_update
* $after_update
* $before_get
* $after_get
* $before_delete
* $after_delete

These are instance variables usually defined at the class level. They are arrays of methods on this class to be called at certain points. An example:

```php
class Book_model extends MY_Model
{
    public $before_create = array( 'timestamps' );
    
    protected function timestamps($book)
    {
        $book['created_at'] = $book['updated_at'] = date('Y-m-d H:i:s');
        return $book;
    }
}
```

**Remember to always always always return the `$row` object you're passed. Each observer overwrites its predecessor's data, sequentially, in the order the observers are defined.**

Observers can also take parameters in their name, much like CodeIgniter's Form Validation library. Parameters are then accessed in `$this->callback_parameters`:

    public $before_create = array( 'data_process(name)' );
    public $before_update = array( 'data_process(date)' );

    protected function data_process($row)
    {
        $row[$this->callback_parameters[0]] = $this->_process($row[$this->callback_parameters[0]]);

        return $row;
    }

Validation
----------

MY_Model uses CodeIgniter's built in form validation to validate data on insert.

You can enable validation by setting the `$validate` instance to the usual form validation library rules array:

    class User_model extends MY_Model
    {
        public $validate = array(
            array( 'field' => 'email', 
                   'label' => 'email',
                   'rules' => 'required|valid_email|is_unique[users.email]' ),
            array( 'field' => 'password',
                   'label' => 'password',
                   'rules' => 'required' ),
            array( 'field' => 'password_confirmation',
                   'label' => 'confirm password',
                   'rules' => 'required|matches[password]' ),
        );
    }

Anything valid in the form validation library can be used here. To find out more about the rules array, please [view the library's documentation](http://codeigniter.com/user_guide/libraries/form_validation.html#validationrulesasarray).

With this array set, each call to `insert()` or `update()` will validate the data before allowing  the query to be run. **Unlike the CodeIgniter validation library, this won't validate the POST data, rather, it validates the data passed directly through.**

You can skip the validation with `skip_validation()`:

    $this->user_model->skip_validation();
    $this->user_model->insert(array( 'email' => 'blah' ));

Alternatively, pass through a `TRUE` to `insert()`:

    $this->user_model->insert(array( 'email' => 'blah' ), TRUE);

Under the hood, this calls `validate()`.

Protected Attributes
--------------------

If you're lazy like me, you'll be grabbing the data from the form and throwing it straight into the model. While some of the pitfalls of this can be avoided with validation, it's a very dangerous way of entering data; any attribute on the model (any column in the table) could be modified, including the ID.

To prevent this from happening, MY_Model supports protected attributes. These are columns of data that cannot be modified.

You can set protected attributes with the `$protected_attributes` array:

    class Post_model extends MY_Model
    {
        public $protected_attributes = array( 'id', 'hash' );
    }

Now, when `insert` or `update` is called, the attributes will automatically be removed from the array, and, thus, protected:

    $this->post_model->insert(array(
        'id' => 2,
        'hash' => 'aqe3fwrga23fw243fWE',
        'title' => 'A new post'
    ));

    // SQL: INSERT INTO posts (title) VALUES ('A new post')

Relationships
-------------

**MY\_Model** now has support for basic _belongs\_to_ and has\_many relationships. These relationships are easy to define:

    class Post_model extends MY_Model
    {
        public $belongs_to = array( 'author' );
        public $has_many = array( 'comments' );
    }

It will assume that a MY_Model API-compatible model with the singular relationship's name has been defined. By default, this will be `relationship_model`. The above example, for instance, would require two other models:

    class Author_model extends MY_Model { }
    class Comment_model extends MY_Model { }

If you'd like to customise this, you can pass through the model name as a parameter:

    class Post_model extends MY_Model
    {
        public $belongs_to = array( 'author' => array( 'model' => 'author_m' ) );
        public $has_many = array( 'comments' => array( 'model' => 'model_comments' ) );
    }

You can then access your related data using the `with()` method:

    $post = $this->post_model->with('author')
                             ->with('comments')
                             ->get(1);

The related data will be embedded in the returned value from `get`:

    echo $post->author->name;

    foreach ($post->comments as $comment)
    {
        echo $message;
    }

Separate queries will be run to select the data, so where performance is important, a separate JOIN and SELECT call is recommended.

The primary key can also be configured. For _belongs\_to_ calls, the related key is on the current object, not the foreign one. Pseudocode:

    SELECT * FROM authors WHERE id = $post->author_id

...and for a _has\_many_ call:

    SELECT * FROM comments WHERE post_id = $post->id

To change this, use the `primary_key` value when configuring:

    class Post_model extends MY_Model
    {
        public $belongs_to = array( 'author' => array( 'primary_key' => 'post_author_id' ) );
        public $has_many = array( 'comments' => array( 'primary_key' => 'parent_post_id' ) );
    }


**Update** foreign_key added 

    class Post_model extends MY_Model
    {
        public $belongs_to = array( 
            'author' => array( 
                'model' => 'author_m',
                'primary_key' => 'post_author_id',
                'foreign_key' => 'author_id'
            ) 
        );
    }



Arrays vs Objects
-----------------

By default, MY_Model is setup to return objects using CodeIgniter's QB's `row()` and `result()` methods. If you'd like to use their array counterparts, there are a couple of ways of customising the model.

If you'd like all your calls to use the array methods, you can set the `$return_type` variable to `array`.

    class Book_model extends MY_Model
    {
        protected $return_type = 'array';
    }

If you'd like just your _next_ call to return a specific type, there are two scoping methods you can use:

    $this->book_model->as_array()
                     ->get(1);
    $this->book_model->as_object()
                     ->get_by('column', 'value');

Soft Delete
-----------

By default, the delete mechanism works with an SQL `DELETE` statement. However, you might not want to destroy the data, you might instead want to perform a 'soft delete'.

If you enable soft deleting, the deleted row will be marked as `deleted` rather than actually being removed from the database.

Take, for example, a `Book_model`:

    class Book_model extends MY_Model { }

We can enable soft delete by setting the `$this->soft_delete` key:

    class Book_model extends MY_Model
    { 
        protected $soft_delete = TRUE;
    }

By default, MY_Model expects a `TINYINT` or `INT` column named `deleted`. If you'd like to customise this, you can set `$soft_delete_key`:

    class Book_model extends MY_Model
    { 
        protected $soft_delete = TRUE;
        protected $soft_delete_key = 'book_deleted_status';
    }

Now, when you make a call to any of the `get_` methods, a constraint will be added to not withdraw deleted columns:

    => $this->book_model->get_by('user_id', 1);
    -> SELECT * FROM books WHERE user_id = 1 AND deleted = 0

If you'd like to include deleted columns, you can use the `with_deleted()` scope:

    => $this->book_model->with_deleted()->get_by('user_id', 1);
    -> SELECT * FROM books WHERE user_id = 1
    
If you'd like to include only the columns that have been deleted, you can use the `only_deleted()` scope:

    => $this->book_model->only_deleted()->get_by('user_id', 1);
    -> SELECT * FROM books WHERE user_id = 1 AND deleted = 1

Built-in Observers
-------------------

**MY_Model** contains a few built-in observers for things I've found I've added to most of my models.

The timestamps (MySQL compatible) `created_at` and `updated_at` are now available as built-in observers:

    class Post_model extends MY_Model
    {
        public $before_create = array( 'created_at', 'updated_at' );
        public $before_update = array( 'updated_at' );
    }

**MY_Model** also contains serialisation observers for serialising and unserialising native PHP objects. This allows you to pass complex structures like arrays and objects into rows and have it be serialised automatically in the background. Call the `serialize` and `unserialize` observers with the column name(s) as a parameter:

    class Event_model extends MY_Model
    {
        public $before_create = array( 'serialize(seat_types)' );
        public $before_update = array( 'serialize(seat_types)' );
        public $after_get = array( 'unserialize(seat_types)' );
    }

Database Connection
-------------------

The class will automatically use the default database connection, and even load it for you if you haven't yet.

You can specify a database connection on a per-model basis by declaring the _$\_db\_group_ instance variable. This is equivalent to calling `$this->db->database($this->_db_group, TRUE)`.

See ["Connecting to your Database"](http://ellislab.com/codeigniter/user-guide/database/connecting.html) for more information.

```php
class Post_model extends MY_Model
{
    public $_db_group = 'group_name';
}
```



Changelog
---------

**Version 3.0.0**
* Added foreign_key to model relationship
