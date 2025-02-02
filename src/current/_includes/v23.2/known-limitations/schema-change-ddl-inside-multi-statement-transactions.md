Schema change [DDL](https://wikipedia.org/wiki/Data_definition_language#ALTER_statement) statements that run inside a multi-statement transaction with non-DDL statements can fail at [`COMMIT`]({% link {{ page.version.version }}/commit-transaction.md %}) time, even if other statements in the transaction succeed.  This leaves such transactions in a "partially committed, partially aborted" state that may require manual intervention to determine whether the DDL statements succeeded.

If such a failure occurs, CockroachDB will emit a CockroachDB-specific error code, `XXA00`, and the following error message:

```
transaction committed but schema change aborted with error: <description of error>
HINT: Some of the non-DDL statements may have committed successfully, but some of the DDL statement(s) failed.
Manual inspection may be required to determine the actual state of the database.
```

{{site.data.alerts.callout_danger}}
If you must execute schema change DDL statements inside a multi-statement transaction, we **strongly recommend** checking for this error code and handling it appropriately every time you execute such transactions.
{{site.data.alerts.end}}

This error will occur in various scenarios, including but not limited to:

- Creating a unique index fails because values aren't unique.
- The evaluation of a computed value fails.
- Adding a constraint (or a column with a constraint) fails because the constraint is violated for the default/computed values in the column.

To see an example of this error, start by creating the following table.

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE TABLE T(x INT);
INSERT INTO T(x) VALUES (1), (2), (3);
~~~

Then, enter the following multi-statement transaction, which will trigger the error.

{% include_cached copy-clipboard.html %}
~~~ sql
BEGIN;
ALTER TABLE t ADD CONSTRAINT unique_x UNIQUE(x);
INSERT INTO T(x) VALUES (3);
COMMIT;
~~~

~~~
pq: transaction committed but schema change aborted with error: (23505): duplicate key value (x)=(3) violates unique constraint "unique_x"
HINT: Some of the non-DDL statements may have committed successfully, but some of the DDL statement(s) failed.
Manual inspection may be required to determine the actual state of the database.
~~~

In this example, the [`INSERT`]({% link {{ page.version.version }}/insert.md %}) statement committed, but the [`ALTER TABLE`]({% link {{ page.version.version }}/alter-table.md %}) statement adding a [`UNIQUE` constraint]({% link {{ page.version.version }}/unique.md %}) failed.  We can verify this by looking at the data in table `t` and seeing that the additional non-unique value `3` was successfully inserted.

{% include_cached copy-clipboard.html %}
~~~ sql
SELECT * FROM t;
~~~

~~~
  x
+---+
  1
  2
  3
  3
(4 rows)
~~~
