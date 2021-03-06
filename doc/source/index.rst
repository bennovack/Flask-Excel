.. flask-excel documentation master file, created by
   sphinx-quickstart on Wed Jan 07 08:46:32 2015.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to Flask-Excel's documentation!
=======================================

:Author: C.W.
:Issues: http://github.com/chfw/Flask-Excel/issues
:License: New BSD License
:Version: |version|
:Generated: |today|

**Flask-Excel** is based on `pyexcel <https://github.com/chfw/pyexcel>`_ and makes it easy to consume/produce information stored in excel files over HTTP protocol as well as on file system. This library can turn the excel data into Pythonic a list of lists, a list of records(dictionaries), dictionaries of lists. And vice versa. Hence it lets you focus on data in Flask based web development, instead of file formats.

The highlighted features are:

#. excel data import into and export from databases
#. turn uploaded excel file directly into Python data struture
#. pass Python data structures as an excel file download
#. provide data persistence as an excel file in server side
#. supports csv, tsv, csvz, tsvz by default and other formats are supported via the following plugins:

.. _file-format-list:

.. table:: A list of file formats supported by external plugins

   ================ ========================================================================
   Plugins          Supported file formats                                      
   ================ ========================================================================
   `xls`_           xls, xlsx(r), xlsm(r)
   `xlsx`_          xlsx
   `ods`_           ods (python 2.6, 2.7)                                       
   `ods3`_          ods (python 2.7, 3.3, 3.4)                                  
   ================ ========================================================================
   
.. _xls: https://github.com/chfw/pyexcel-xls
.. _xlsx: https://github.com/chfw/pyexcel-xlsx
.. _ods: https://github.com/chfw/pyexcel-ods
.. _ods3: https://github.com/chfw/pyexcel-ods3
.. _text: https://github.com/chfw/pyexcel-text

This library makes infomation processing involving various excel files as easy as processing array, dictionary when processing file upload/download, data import into and export from SQL databases, information analysis and persistence. It uses **pyexcel** and its plugins: 1) to provide one uniform programming interface to handle csv, tsv, xls, xlsx, xlsm and ods formats. 2) to provide one-stop utility to import the data in uploaded file into a database and to export tables in a database as excel files for file download 3) to provide the same interface for information persistence at server side: saving a uploaded excel file to and loading a saved excel file from file system.

Quick start
------------

A minimal application may look like this::

    from flask import Flask, request, jsonify
    from flask.ext import excel
    
    app=Flask(__name__)
    
    @app.route("/upload", methods=['GET', 'POST'])
    def upload_file():
        if request.method == 'POST':
            return jsonify({"result": request.get_array(field_name='file')})
        return '''
        <!doctype html>
        <title>Upload an excel file</title>
        <h1>Excel file upload (csv, tsv, csvz, tsvz only)</h1>
        <form action="" method=post enctype=multipart/form-data><p>
        <input type=file name=file><input type=submit value=Upload>
        </form>
        '''
    
    @app.route("/download", methods=['GET'])
    def download_file():
        return excel.make_response_from_array([[1,2], [3, 4]], "csv")

    # insert database related code here
    
    if __name__ == "__main__":
        app.run()


The tiny application exposes two urls: one for file upload and the other for file donload. The former url presents a simple file upload html and responds back in json with the content of the uploaded file. Here is an `example file <https://github.com/chfw/Flask-Excel/blob/master/examples/example_for_upload.csv>` for testing but you can upload any other excel file. The file upload handler uses `request.get_array` to parse the uploaded file and gets an array back. The parameter **file** is coded in the html form::

    <input ... name=file>

.. warning::
   If 'field_name' was not specified, for example `request.get_array('file')` in upload_file() function, your browser would display "Bad Request: The browser (or proxy) sent a request that this server could not understand."

The latter simply throws back a csv file whenever a http request is made to http://localhost:50000/download/. `excel.make_response_from_array` takes a list of lists and a file type as parameters and sets up the mime type of the http response. If you would like to give 'tsvz' a go, please change "csv" to "tsvz".

More excel file formats
------------------------
    
The example application understands csv, tsv and its zipped variants: csvz and tsvz. If you would like to expand the list of supported excel file formats (see :ref:`file-format-list`) for your own application, you could include one or all of the following import lines right after **Flask-Excel** is imported::

    import pyexcel.ext.xls
    import pyexcel.ext.xlsx
    import pyexcel.ext.ods

Data import and export
-----------------------------

Continue with the previous example, the data import and export will be explained. You can copy the following code in their own appearing sequence and paste them after the place holder::

    # insert database related code here

Alernatively, you can find the complete example on `github <https://github.com/chfw/Flask-Excel/blob/master/examples/database_example.py>`_

Now let's add the following imports first::

    from flask.ext.sqlalchemy import SQLAlchemy # sql operations
    import pyexcel.ext.xls # import it to be able to handle xls file format

Now configure the database connection. Sqllite will be used and **tmp.db** will be used and can be found in your current working directory::
    
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///tmp.db'
    db = SQLAlchemy(app)

And paste some models from Flask-SQLAlchemy's documentation::

    class Post(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        title = db.Column(db.String(80))
        body = db.Column(db.Text)
        pub_date = db.Column(db.DateTime)
    
        category_id = db.Column(db.Integer, db.ForeignKey('category.id'))
        category = db.relationship('Category',
            backref=db.backref('posts', lazy='dynamic'))
    
        def __init__(self, title, body, category, pub_date=None):
            self.title = title
            self.body = body
            if pub_date is None:
                pub_date = datetime.utcnow()
            self.pub_date = pub_date
            self.category = category
    
        def __repr__(self):
            return '<Post %r>' % self.title
    
    class Category(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        name = db.Column(db.String(50))
    
        def __init__(self, name):
            self.name = name
    
        def __repr__(self):
            return '<Category %r>' % self.name

Now let us create the tables in the database::

    db.create_all()

Write up the view functions for data import::

    @app.route("/import", methods=['GET', 'POST'])
    def doimport():
        if request.method == 'POST':
            def category_init_func(row):
                c = Category(row['name'])
                c.id = row['id']
                return c
            def post_init_func(row):
                c = Category.query.filter_by(name=row['category']).first()
                p = Post(row['title'], row['body'], c, row['pub_date'])
                return p
            request.save_book_to_database(field_name='file', session=db.session,
                                          tables=[(Category, category_init_func),
                                                  (Post, post_init_func)])
            return "Saved"
        return '''
        <!doctype html>
        <title>Upload an excel file</title>
        <h1>Excel file upload (xls, xlsx, ods please)</h1>
        <form action="" method=post enctype=multipart/form-data><p>
        <input type=file name=file><input type=submit value=Upload>
        </form>
        '''

Write up the view function for data export::

    @app.route("/export", methods=['GET'])
    def doexport():
        return excel.make_response_from_tables(db.session, [Category, Post], "xls")


Then run the example again. Visit http://localhost:5000/import and upload `sample-data.xls <https://github.com/chfw/Flask-Excel/blob/master/sample-data.xls>`_ . Then visit http://localhost:5000/export to download the data back.

Export filtered query sets
-----------------------------

Previous example shows you how to dump one or more tables over http protocol. Hereby, let's look at how to turn a query sets into an excel sheet. You can
pass a query sets and an array of selected column names to :meth:`~flask_excel.make_response_from_query_sets` and generate an excel sheet from it::

    @app.route("/custom_export", methods=['GET'])
    def docustomexport():
        query_sets = Category.query.filter_by(id=1).all()
        column_names = ['id', 'name']
        return excel.make_response_from_query_sets(query_sets, column_names, "xls")

Then visit http://localhost:5000/custom_export to download the data

All supported data types
--------------------------

The example application likes to have array but it is not just about arrays. Here is table of functions for all supported data types:

=========================== ======================================================= ==================================================
data structure              from file to data structures                            from data structures to response
=========================== ======================================================= ==================================================
dict                        :meth:`~flask_excel.ExcelRequest.get_dict`              :meth:`~flask_excel.make_response_from_dict`
records                     :meth:`~flask_excel.ExcelRequest.get_records`           :meth:`~flask_excel.make_response_from_records`
a list of lists             :meth:`~flask_excel.ExcelRequest.get_array`             :meth:`~flask_excel.make_response_from_array`
dict of a list of lists     :meth:`~flask_excel.ExcelRequest.get_book_dict`         :meth:`~flask_excel.make_response_from_book_dict`
:class:`pyexcel.Sheet`      :meth:`~flask_excel.ExcelRequest.get_sheet`             :meth:`~flask_excel.make_response`
:class:`pyexcel.Book`       :meth:`~flask_excel.ExcelRequest.get_book`              :meth:`~flask_excel.make_response`
database table              :meth:`~flask_excel.ExcelRequest.save_to_database`      :meth:`~flask_excel.make_response_from_a_table` 
a list of database tables   :meth:`~flask_excel.ExcelRequest.save_book_to_database` :meth:`~flask_excel.make_response_from_tables` 
a database query sets                                                               :meth:`~flask_excel.make_response_from_query_sets`
=========================== ======================================================= ==================================================

See more examples of the data structures in :ref:`pyexcel documentation<pyexcel:a-list-of-data-structures>`

API Reference
---------------

**Flask-Excel** attaches **pyexcel** functions to **Request** class.

.. module:: flask_excel

.. autoclass:: ExcelRequest

   .. method:: get_sheet(field_name=None, sheet_name=None, **keywords)

      :param field_name: the file field name in the html form for file upload
      :param sheet_name: For an excel book, there could be multiple sheets. If it is left
                         unspecified, the sheet at index 0 is loaded. For 'csv', 'tsv' file,
                         *sheet_name* should be None anyway.
      :param keywords: additional keywords to :meth:`pyexcel.get_sheet`
      :returns: A sheet object

      The following html form, the *field_name* should be "file"::
    
          <!doctype html>
          <title>Upload an excel file</title>
          <h1>Excel file upload (csv, tsv, csvz, tsvz only)</h1>
          <form action="" method=post enctype=multipart/form-data><p>
          <input type=file name=file><input type=submit value=Upload>
          </form>

   .. method:: get_array(field_name=None, sheet_name=None, **keywords)

      :param field_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param sheet_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param keywords: additional keywords to pyexcel library
      :returns: a two dimensional array, a list of lists

   .. method:: get_dict(field_name=None, sheet_name=None, name_columns_by_row=0, **keywords)

      :param field_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param sheet_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param name_columns_by_row: uses the first row of the sheet to be column headers by default.
      :param keywords: additional keywords to pyexcel library
      :returns: a dictionary of the file content

   .. method:: get_records(field_name=None, sheet_name=None, name_columns_by_row=0, **keywords)

      :param field_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param sheet_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param name_columns_by_row: uses the first row of the sheet to be record field names by default.
      :param keywords: additional keywords to pyexcel library
      :returns: a list of dictionary of the file content

   .. method:: get_book(field_name=None, **keywords)

      :param field_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param keywords: additional keywords to pyexcel library
      :returns: a two dimensional array, a list of lists

   .. method:: get_book_dict(field_name=None, **keywords)

      :param field_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param keywords: additional keywords to pyexcel library
      :returns: a two dimensional array, a list of lists

   .. method:: save_to_database(field_name=None, session=None, table=None, initializer=None, mapdict=None **keywords)

      :param field_name: same as :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param session: a SQLAlchemy session						
      :param table: a database table 
      :param initializer: a custom table initialization function if you have one
      :param mapdict: the explicit table column names if your excel data do not have the exact column names
      :param keywords: additional keywords to :meth:`pyexcel.Sheet.save_to_database`

   .. method:: save_book_to_database(field_name=None, session=None, tables=None, **keywords)

      :param field_name: save as  :meth:`~flask_excel.ExcelRequest.get_sheet`
      :param session: a SQLAlchemy sessio
      :param tables: a list of database tables
      :param initializers: a list of model initialization functions.
      :param mapdicts: a list of explicit table column names if your excel data sheets do not have the exact column names
      :param keywords: additional keywords to :meth:`pyexcel.Book.save_to_database`


Response methods
-----------------

   .. method:: make_response(pyexcel_instance, file_type, status=200)

      :param pyexcel_instance: :class:`pyexcel.Sheet` or :class:`pyexcel.Book`
      :param file_type: one of the following strings:
                        
                        * 'csv'
                        * 'tsv'
                        * 'csvz'
                        * 'tsvz'
                        * 'xls'
                        * 'xlsx'
                        * 'xlsm'
                        * 'ods'
                          
      :param status: unless a different status is to be returned.
            
   .. method:: make_response_from_array(array, file_type, status=200)

      :param array: a list of lists
      :param file_type: same as :meth:`~flask_excel.make_response`
      :param status: same as :meth:`~flask_excel.make_response`
            
   .. method:: make_response_from_dict(dict, file_type, status=200)

      :param dict: a dictinary of lists
      :param file_type: same as :meth:`~flask_excel.make_response`
      :param status: same as :meth:`~flask_excel.make_response`
            
   .. method:: make_response_from_records(records, file_type, status=200)

      :param records: a list of dictionaries
      :param file_type: same as :meth:`~flask_excel.make_response`
      :param status: same as :meth:`~flask_excel.make_response`
            
                
   .. method:: make_response_from_book_dict(book_dict, file_type, status=200)

      :param book_dict: a dictionary of two dimensional arrays
      :param file_type: same as :meth:`~flask_excel.make_response`
      :param status: same as :meth:`~flask_excel.make_response`

   .. method:: make_response_from_a_table(session, table, file_type status=200)

      Produce a single sheet Excel book of *file_type*

      :param session: SQLAlchemy session
      :param table: a SQLAlchemy table
      :param file_type: same as :meth:`~flask_excel.make_response`
      :param status: same as :meth:`~flask_excel.make_response`

    .. method:: make_response_from_query_sets(query_sets, column_names, file_type status=200)

      Produce a single sheet Excel book of *file_type* from your custom database queries

      :param query_sets: a query set
      :param column_names: a nominated column names. It could not be None, otherwise no data is returned.
      :param file_type: same as :meth:`~flask_excel.make_response`
      :param status: same as :meth:`~flask_excel.make_response`

   .. method:: make_response_from_tables(session, tables, file_type status=200)

      Produce a multiple sheet Excel book of *file_type*. It becomes the same
      as :meth:`~flask_excel.make_response_from_a_table` if you pass *tables*
      with an array that has a single table
      
      :param session: SQLAlchemy session
      :param tables: SQLAlchemy tables
      :param file_type: same as :meth:`~flask_excel.make_response`
      :param status: same as :meth:`~flask_excel.make_response`

                
Indices and tables
--------------------

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

