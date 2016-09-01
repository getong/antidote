---
title: API
last_updated: August 29, 2016
tags: [api]
summary: ""
sidebar: mydoc_sidebar
permalink: rawapi.html
folder: mydoc
---

This page describes the raw API of Antidote. These are erlang methods and clients can invoke these API methods via RPC. A better way for client applications to interact with Antidote is to use the [protocol buffer interface](/api.html).

Antidote API
-----------

A unit of operation in Antidote is a transaction. A client should first start a transaction, then read and/or update multiple objects, and then commit the transaction.

There are two types of transactions: interactive transactions and static transactions.

### Interactive transactions ###

In interactive transaction, a client can start the transaction and execute many updates and reads and then commit the transactions. The interface of interactive transaction is:

    type bound_object() = {key(), crdt_type(), bucket()}.
    type snapshot_time() = vectorclock() | ignore.

    start_transation(snapshot_time(), properties()) -> {ok, txid()} | {error, reason()}.

    update_objects([{bound_object(), operation(), op_param()}], txid()) -> ok | {error, reason()}.

    read_objects([bound_object()], TxId) -> {ok, [term()]}.

    commit_transaction(txid()) -> {ok, vectorclock()} | {error, reason()}.

#### Example ####
     CounterObj = {my_counter, antidote_crdt_counter, my_bucket},
     {ok, TxId} = rpc:call(Node, antidote, start_transaction, [ignore, []]),
     {ok, [CounterVal]} = rpc:call(Node, antidote, read_objects, [[CounterObjm], TxId]),
     ok = rpc:call(Node, antidote, update_objects, [[{CounterObj, increment, 1}], TxId]),
     {ok, CT} = rpc:call(Node, antidote, commit_transaction, [TxId]),
     %% Start a new transaction
     {ok, TxId2} = rpc:call(Node, antidote, start_transaction, [CT, []]),
     ....

### Static transactions ###
In static transaction, client can issue a single call with either update to multiple objects or read to multiple objects. It cannot read and update in the same transaction. The interface of static transaction is:

     type bound_object() = {key(), crdt_type(), bucket()}.
     type snapshot_time() = vectorclock() | ignore.

     update_objects(snapshot_time(), properties(), [{bound_object(), operation(), op_param()}]) ->
                {ok, vectorclock()} | {error, reason()}.

     read_objects(snapshot_time, properties(), [bound_object()]) -> {ok, [term()], vectorclock()}.

#### Example ####
    CounterObj = {my_counter, antidote_crdt_counter, my_bucket},
    SetObj = {my_set, antidote_crdt_orset, my_bucket},
    {ok, CT1} = rpc:call(Node, antidote, update_objects, [ignore, [], [{CounterObj, increment, 1}]]),
    {ok, Result, CT2} = rpc:call(Node, antidote, read_objects, [CT1, [], [CounterObj, SetObj]]),
    [CounterVal, SetVal] = Result.

### Antidote CRDTs ###

Antidote has a library of crdts. The interface of these CRDTs specify what are the operations and the parameters that can be given in update_objects method. Here we specify the supported {operation(), op_param()} pair for each of the supported crdts.

##### antidote_crdt_counter #####
    {increment, integer}
    {decrement, integer()}

##### antidote_crdt_orset #####
    {add, term()}
    {remove, term()}
    {add_all, [term()]}
    {remove_all, [term()]}

##### antidote_crdt_gset #####
    {add, {term(), actor()}}
    {remove, {term(), actor()}}
    {add_all, {[term()], actor()}}
    {remove_all, {[term()], actor()}}

##### antidote_crdt_lwwreg #####
    {assign, {term(), non_neg_integer()}}
    {assign, term()}.

##### antidote_crdt_map #####
    {update, {[map_field_update() | map_field_op()], actorordot()}}.

    -type actorordot() :: riak_dt:actor() | riak_dt:dot().
    -type map_field_op() ::  {remove, field()}.
    -type map_field_update() :: {update, field(), crdt_op()}.
    -type crdt_op() :: term(). %% Valid riak_dt updates
    -type field() :: term()

##### antidote_crdt_mvreg #####
    {assign, {term(), non_neg_integer()}}
    {assign, term()}
    {propagate, {term(), non_neg_integer()}}

##### antidote_crdt_rga #####
    {addRight, {any(), non_neg_integer()}}
    {remove, non_neg_integer()}