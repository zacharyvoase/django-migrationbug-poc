# Django 1.8 Migration Bug Proof-of-Concept

I've identified a bug in migrations in Django 1.8 w/ Postgres (that definitely
wasn't present in v1.7).


## Bug Description

If you have a field like:

    name = models.CharField(..., db_index=True)

And you add a unique constraint without removing the `db_index=True`:

    name = models.CharField(..., db_index=True, unique=True)

The auto-migration you get from that looks like:

    migrations.AlterField(
        model_name='person',
        name='name',
        field=models.CharField(db_index=True, unique=True, max_length=255, blank=True),
    )

When Django tries to run this migration you get:

```
Operations to perform:
  Synchronize unmigrated apps: staticfiles, messages
  Apply all migrations: admin, contenttypes, migration_poc_app, auth, sessions
Synchronizing apps without migrations:
  Creating tables...
    Running deferred SQL...
  Installing custom SQL...
Running migrations:
  Rendering model states... DONE
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying migration_poc_app.0001_initial... OK
  Applying migration_poc_app.0002_auto_20160114_2114...Traceback (most recent call last):
  File "manage.py", line 10, in <module>
    execute_from_command_line(sys.argv)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/core/management/__init__.py", line 354, in execute_from_command_line
    utility.execute()
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/core/management/__init__.py", line 346, in execute
    self.fetch_command(subcommand).run_from_argv(self.argv)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/core/management/base.py", line 394, in run_from_argv
    self.execute(*args, **cmd_options)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/core/management/base.py", line 445, in execute
    output = self.handle(*args, **options)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/core/management/commands/migrate.py", line 222, in handle
    executor.migrate(targets, plan, fake=fake, fake_initial=fake_initial)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/migrations/executor.py", line 110, in migrate
    self.apply_migration(states[migration], migration, fake=fake, fake_initial=fake_initial)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/migrations/executor.py", line 148, in apply_migration
    state = migration.apply(state, schema_editor)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/migrations/migration.py", line 115, in apply
    operation.database_forwards(self.app_label, schema_editor, old_state, project_state)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/migrations/operations/fields.py", line 201, in database_forwards
    schema_editor.alter_field(from_model, from_field, to_field)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/backends/base/schema.py", line 484, in alter_field
    old_db_params, new_db_params, strict)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/backends/postgresql_psycopg2/schema.py", line 113, in _alter_field
    self.execute(like_index_statement)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/backends/base/schema.py", line 111, in execute
    cursor.execute(sql, params)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/backends/utils.py", line 79, in execute
    return super(CursorDebugWrapper, self).execute(sql, params)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/backends/utils.py", line 64, in execute
    return self.cursor.execute(sql, params)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/utils.py", line 98, in __exit__
    six.reraise(dj_exc_type, dj_exc_value, traceback)
  File "/Users/zack/src/migration_poc/venv/lib/python2.7/site-packages/django/db/backends/utils.py", line 64, in execute
    return self.cursor.execute(sql, params)
django.db.utils.ProgrammingError: relation "migration_poc_app_person_name_32437daa2f5b1fe9_like" already exists
```

**N.B.:** If you remove the `db_index=True` from the `AlterField(...)` call,
the migration succeeds, even if the field definition in the model still has
`db_index=True`.


## Repro Instructions

This repo includes a minimal Django project that produces the error.

1. Create a virtualenv and install Django and psycopg2:

   ```
   virtualenv venv
   . venv/bin/activate
   pip install -r requirements.txt
   ```

2. Install Postgres, e.g. from [Postgres.app](http://postgresapp.com)
3. Edit `migration_poc/settings.py` to point to your local Postgres instance
4. Create a `migration_poc` database:

   ```
   createdb migration_poc
   ```

5. Run the migration:

   ```
   python manage.py migrate
   ```

   This should produce the error.

You can make changes to the auto-migration and models.py, then just run `dropdb
migration_poc; createdb migration_poc` to reset the DB and try migrating again.
This is how I found out that removing the `db_index=True` param from
`AlterField` side-steps the issue.
