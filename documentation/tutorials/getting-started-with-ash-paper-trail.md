# Getting started with AshPaperTrail

First, add the dependency to your `mix.exs`

```elixir
{:ash_paper_trail, "~> 0.1.2-rc.0"}
```

Then, add the `AshPaperTrail.Resource` extension to any resource you would like to version and configure the change tracking mode

```elixir
use Ash.Resource,
  domain: MyDomain,
  extensions: [
    AshPaperTrail.Resource
  ]

  paper_trail do
    change_tracking_mode :changes_only # default is :snapshot
    store_action_name? true # default is false
    ignore_attributes [:inserted_at, :updated_at] # the primary keys are always ignored
  end
```

This will generate the version resource automatically and add them to your domain. The autogenerated resource will be named `Version` under the namespace of the original resource and will belong to the original resource. For example, if your original resource is `MyApp.Post` the autogenerated resource will be `MyApp.Post.Version`. `Post` has_many `paper_trail_versions` and `Version` belong_to `source_version`

Then for each Domain for versioned resources, you need the versioned resource to the Domain of the original non-versioned resource's domain. There are a few ways to achieve this.

Add the `AshPaperTrail.Domain` extension to your domain.

```elixir
use Ash.Domain,
  extensions: [
    AshPaperTrail.Domain
  ]
```

In some use cases you'll need each resource to be a part of the domain--if your versioned resources are part of a graphql. If that's the case, add each versioned resource explicitly to your domain's resources.

```
use Ash.Domain

resources do
  resource MyApp.Post
  resource MyApp.Post.Version
end
```

## Destroy Actions

If you are using `AshPostgres`, and you want to support destroy actions, you need to do one of two things:

1. use something like `AshArchival` in conjunction with this resource to ensure that destroy actions are `soft?` and do not actually result in row deletion

2. configure `AshPaperTrail` not to create references, via:

```elixir
paper_trail do
  reference_source? false
end
```

## Attributes

By default, attribute values are stored in the `changes` attribute. This is to protect you over time as your resources change. However, if there are attributes that you are confident will not change,
you can create attributes for them on the version resource, like so:

```elixir
paper_trail do
  attributes_as_attributes [:organization_id, :author_id]
end
```

This will make your version resource have `foo` and `bar` attributes (they will still show up in `changes`), i.e

```elixir
%ThingVersion{foo: "foo", bar: "bar", changes: %{"foo" => "foo", "bar" => "bar"}}
```

## Change Tracking Modes

Valid options are `:snapshot` and `:changes_only` and `:full_diff`.

### Snapshots

`:snapshot` will json dump the contents of every attribute whether they changed or not.

`{ subject: "new subject", body: "unchanged body", author: { name: "bob"}}`

### Changes Only

`:changes_only` will json dump the contents of only the attributes that have changed.
Note if any part of an embedded attribute and array of embedded attributes, changes then the entire top level attribute is dumped.

`{ subject: "new subject" }`

### Full Diff

`:full_diff` will json dump the contents of each attribute.

`{ subject: { from: "subject", to: "new subject" }, body: { unchanged: "unchanged_body" }}, author: { changes: { unchanged: "bob" }}`

## Associating Versions with Actors

You can record the actor who made the change by declaring one or more resources that can be actors.

```
paper_trail do
  belongs_to_actor :user, MyApp.Accounts.User, domain: MyApp.Accounts
  belongs_to_actor :news_feed, MyApp.Accounts.NewsFeed, domain: MyApp.Accounts
end
```

Each `belongs_to_actor` will create a `belongs_to` relationship with the given name destination. When creating a new version, if the actor on the action is set and matches the resource type, the version will be related to the actor. If your actors are polymorphic or varying types, declare a belongs_to_actor for each type.

A reference is also created with `on_delete: :nilify` and `on_update: :update`

If you need a more complex relationship or your actor is not a resource (e.g. String), the actor is always set on Version create and you can store it by adding `:on_create` `change` in a mixin.

## Multitenancy

If your resource uses multitenancy, then the strategy, attribute, and parse_attribute options (if any) will be applied to the version resource. If using the attribute strategy you will need to ensure this is also an attribute on the version using the `attributes_as_attributes` option (described above) or via a mixin (described below)

## Enriching the Versions resource

If you want to do something like exposing your versions resource over your graphql, you can use the `mixin` and `version_extensions` options.

For example:

```elixir
paper_trail do
  mixin {MyApp.MyResource.PaperTrailMixin, :graphql, [:my_resource_version]}
  version_extensions extensions: [AshGraphql.Resource]
end
```

And then you can define a module like so:

```elixir
defmodule MyApp.MyResource.PaperTrailMixin do

  def graphql(type) do
    quote do
      graphql do
        type unquote(type)

        queries do
          list :list_versions, action: :read
        end
      end
    end
  end
end
```