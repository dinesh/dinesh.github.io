---
layout: post
title: Advisory Locks in Postgresql
---

PostgreSQL provides various lock modes to control concurrent access to data in tables. Advisory locks provide a convenient way to obtain a lock from PostgreSQL that is completely application enforced, and will not block writes to the table.

Imagine you have a scheduled task that runs in the background, mutates the database and sends information off to another 3rd party service. We can use PostgreSQL Advisory Locks to guarantee that the program cannot cause any unexpected behavior if ran multiple times concurrently. Concurrency is fun, but surprise concurrency can be brutal! In our case it's beneficial to have a guarantee that the program in question can never run concurrently.

Application Enforced Locks
--------------------------

Throughout this post we will explore Postgres advisory locks, which are application enforced database locks. Advisory locks can be acquired at the session level and at the transaction level and release as expected when a session ends or a transaction completes. So what does it mean to have an application enforced database lock? Essentially, when your process starts up, you can acquire a lock through Postgres and then release it when the program exits. In this way, we have a guarantee that the program cannot be running concurrently, as it will not be able to acquire the lock at startup, in which case you can just exit.

The benefit of this is that the tables are never actually locked for writing, so the main application can behave as normal and users will never notice anything is happening in the background.

Here is the syntax for obtaining the lock:

  SELECT pg_try_advisory_lock(1);

Now, there's a few things happening in that statement so lets examine it.

* `pg_try_advisory_lock(key);` is the Postgres function that will try to obtain the lock. If the lock is successfully obtained, Postgres will return `'t'`, otherwise `'f'` if you failed you obtain the lock.

* `pg_try_advisory_lock(key)` takes an argument which will become the identifier for the lock. So, for example, if you were to pass in `1`, and another session tried to call `SELECT pg_try_advisory_lock(1)` it would return `'f'`, however the other session could obtain `SELECT pg_try_advisory_lock(2)`.

* `pg_try_advisory_lock(key)` will not wait until it can obtain the lock, it will return immediately with `'t'` or `'f'`.


Implementation
---------------

So how would we go about using this in Ruby?

  def obtained_lock?
      connection.select_value('select pg_try_advisory_lock(1);') == 't'
  end
  
We can grab our ActiveRecord connection and call `#select_value` in order get back a `'t'` or `'f'` value. A simple equality check let's us know whether or not we have obtained the lock, and if we haven't we can choose to bail and exit the program.


```
class LockObtainer
  def lock_it_up
    exclusive do
      # do important stuff here
    end
  end

  private

  def exclusive
    if obtained_lock?
      begin
        yield
      ensure
        release_lock
      end
    end
  end

  def obtained_lock?
    connection.select_value('select pg_try_advisory_lock(1);') == 't'
  end

  def release_lock
    connection.execute 'select pg_advisory_unlock(1);'
  end

  def connection
    ActiveRecord::Base.connection
  end
end
```


Additional Information
-----------------------

There are a few interesting things about Advisory Locks:

* They are reference counted, so you can obtain a lock N times, but must release it N times for another process to acquire it.

* Advisory Locks can be acquired at the session level or the transaction level, meaning if acquired within a transaction, an Advisory Lock will be automatically released for you. Outside of a transaction, you must manually release the lock, or end your session with PostgreSQL.

* Calling `SELECT pg_advisory_unlock_all()` will unlock all advisory locks currently held by you in your session.

* Calling `SELECT pg_advisory_unlock(n)` will release one lock reference for the id `n`.
