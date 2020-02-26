---
layout: post
title: "Expose Postgresql Stored Functions as Hasura Mutations"
date: 2020-02-25
categories: [hasura, plpgsql, postgresql]
---

Hasura does not provide a direct mechanism to call postgresql stored functions as mutations at the current version (1.2-beta),
but you can combine some easy tricks to archive that.

## Requirements

1. Expose arbitrary user defined functions as mutations
2. Accept arbitrary parameters
3. Return arbitrary results
4. Control which functions are exposed by graphql

## Proposed Solution

1. Define a table to list exposed functions
2. Define a table to enqueue function calls
3. Define a trigger to call actual functions
4. Configure hasura with proper presets

![fig1](/images/hasura-action-pg.png)

## Implementation

**1. Create a table for the names of the functions that will be exposed by GraphQL**

```sql
--    This table saves actions types.
--    Any action inserted in action_journal must be registerded here.
CREATE TABLE action (
    id_ text not null primary key
);
```

**2. Create a table to insert function calls (mutation)**

```sql
--    This table saves actions. A trigger on insert will
--    dispatch logic associated with the action.
--    Action Functions Contract:
--       Signatuire: 
--         FUNCTION action_{one of action.id_}(journalId bigint, userId text, request jsonb) RETURNS jsonb
--       Returns:
--         { _status: success | error, _message?, ...rest }
--
CREATE TABLE action_journal(
    id_ bigserial primary key,
    ts_ timestamp not null default now(),
    user_id_ text not null,
    action_id_ text not null references action(id_),
    request_ jsonb not null,
    response_ jsonb
);
```

**3. Define a trigger to call actual functions on insert on action_journal**

```sql
--    Dispatch actions to the actual function based on action_journal.action_id_.
--    Important: Due to postgresql limitations about transaction handling in triggers,
--               any exception raised from the function will propagate to the caller
--               and the transaction will be rolled back.
CREATE OR REPLACE FUNCTION action_dispatcher_trigger() 
RETURNS trigger AS $BODY$
DECLARE
    response jsonb;
BEGIN
    EXECUTE 'SELECT action_' || NEW.action_id_ || '($1, $2, $3)' 
        INTO response
        USING NEW.id_, NEW.user_id_, NEW.request_;
    NEW.response_ = response;
    RETURN NEW;
END;
$BODY$
LANGUAGE plpgsql VOLATILE;

CREATE TRIGGER dispatcher
    BEFORE INSERT
    ON action_journal
    FOR EACH ROW
    EXECUTE PROCEDURE action_dispatcher_trigger();

```

**4. Configure hasura**

1. Expose table action_journal
2. Set insert permissions as needed
3. Configure insert presets to match user_id_ with session var x-hasura-user-id
        ![fig2](/images/hasura-presets.png)


## Test

Ok, now we can define our own function, register it in actions and call it from GraphQL

**1. Define our function**

```sql
-- Simple function to do something stupid.
-- request: { a, b }
CREATE OR REPLACE FUNCTION action_sum(action_id bigint, user_id text, request jsonb)
RETURNS jsonb AS $BODY$
DECLARE
    result numeric;
BEGIN
    result = (request->>'a')::numeric + (request->>'b')::numeric;
    RETURN jsonb_build_object(
        '_status', 'success',
        'result', result
    );
END;
$BODY$
LANGUAGE plpgsql VOLATILE;

```

**2. Register our function as public**

Expose our function

```sql
INSERT INTO action('sum');
```

The trigger will prepend 'action_' and call action_sum

**3. Call our function as a graphql mutation**

```graphql
mutation POST_ACTION($object : action_journal_insert_input!) {
    action: insert_action_journal(objects: [$object]) {
      returning {
        response_
      }
    }
}

variables {
    object: {
        action_id_: "sum",
        request_: { a: 10, b: 20 }
    }
}
```

**Result:**

```graphql
{
    data: {
        action: {
            returning: [
                {
                    response_: {
                        _status: "success",
                        result: 30
                    }
                }
            ]
        }
    }
}
```

To add more functions you just need to insert the name in table action and create the function in postgresql with 'action_' prefix.












