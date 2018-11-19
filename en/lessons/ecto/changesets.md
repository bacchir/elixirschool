---
version: 1.0.0
title: Changesets
---

In order to insert, update or delete data from the database, `Ecto.Repo.insert/2`, `update/2` and `delete/2` require a changeset as their first parameter.
But what exactly are changesets?

A familiar task for almost every developer is checking input data for potential errors — we want to make sure that data is in the right state, before we attempt to use it for our purposes.

Ecto provides a complete solution for working with data changes in the form of the `Changeset` module and data structure.
In this lesson we're going to explore this functionality and learn how to verify data's integrity, before we persist it to the database.

{% include toc.html %}

## Creating your first changeset

Let's look at an empty `%Changeset{}` struct:

```elixir
iex> %Ecto.Changeset{}
#Ecto.Changeset<action: nil, changes: %{}, errors: [], data: nil, valid?: false>
```

As you can see, it has some potentially useful fields, but they are all empty.

For a changeset to be truly useful, when we create it, we need to provide a blueprint of what the data is like.
What better blueprint for our data than the schemas we've created the define our fields and types?

To save us some time, let's re-use the schema we created in the previous lesson:

```elixir
defmodule Example.Person do
  use Ecto.Schema

  schema "people" do
    field :name, :string
    field :age, :integer, default: 0
  end
end
```

### `cast/4`

To create a changeset using the `Example.Person` schema, we are going to use [`Ecto.Changeset.cast/4`](https://hexdocs.pm/ecto/Ecto.Changeset.html#cast/4).
The first parameter should be our starting data which for us will be an empty `%Person{}` but could be an existing record.
The secord parameter `cast/4` expects is the data changes as a map.
Our third parameter is a list of fields we're accepting values for, fields not included in this list will be ignored.
The foruth and final parameter are changeset options which we'll explore later, for now we'll focus on the first three:

```elixir
iex> alias Example.Person
iex> Ecto.Changeset.cast(%Person{}, %{"name" => "Jack"}, [:age, :name])
#Ecto.Changeset<
 action: nil,
 changes: %{name: "Jack"},
 errors: [],
 data: #Person<>,
 valid?: true
>
 ```

We can see that we've created a new changeset and this time it's populated with values, most importantly: `:changes` and `:valid?`.

Before we go too much further let's quickly take a look at how permitted fields work:

 ```elixir
iex> Ecto.Changeset.cast(%Person{}, %{"email" => "jack@example.com", "name" => "Jack"}, [:name])
#Ecto.Changeset<
 action: nil,
 changes: %{name: "Jack"},
 errors: [],
 data: #Person<>,
 valid?: true
>
```

What we see is the same valid changeset from before with no mention of the `"email"` value because it was not explicitly permitted.

### `change/2`

If the source of the data change is truthworthy `change/2` can be a viable alternative to `cast/4`.
The primary difference between these two functions is that `change/2` does no filtering or casting of values.

To quickly demonstrate this let's assume a person's birthday is entered and age is computed from that.
Since we perform this calculation ourelves we can trust that the value will be correctly formatted and cast:

```elixir
%Person{}
|> Ecto.Changeset.cast(%Person{}, %{"name" => "Jack"}, [:name])
|> Ecto.Changeset.change(%{age: 30})

#Ecto.Changeset<
 action: nil,
 changes: %{age: 30, name: "Jack"},
 errors: [],
 data: #Person<>,
 valid?: true
>
```

## Validations

We've learned how to create changesets and apply changes to them but how do we validate those changes?

If we take our previous examples which lack validation, any changes to a person's name is considered valid which means a blank string counts:

```elixir
iex> Ecto.Changeset.cast(%Person{}, %{"name" => ""}, [:name])
#Ecto.Changeset<
 action: nil,
 changes: %{name: ""},
 errors: [],
 data: #Person<>,
 valid?: true
>
```

This won't work for a real application so let's look at how we can handle this with Ecto.
Lucky for us, as with many of the Elixir libraries, Ecto comes with a slew of awesome features one of which is built-in validaitons.

Before we dive into validations let's make a quick change to our `Exmaple.Person` module.
We'll be using `Ecto.Changeset` a lot going forward so let's import it into our module:

```elixir
defmodule Example.Person do
  use Ecto.Schema

  import Ecto.Changeset

  schema "people" do
    field :name, :string
    field :age, :integer, default: 0
  end
end
```

Now we can use the `cast/4`, `change/2`, and the built-in validators directly.

### `validate_required/3`

A common practice is to encapsulate the changeset functionality and validation in one or more functions for a given schema.
For our example we'll stick with the more common `changeset/2` function in our `Example.Person` module.
This function will take a struct, a `%Person{}` new or existing, and a map of changes.

We know we don't want to allow blank names so let's create our new `changeset/2` function to use a `cast/4` and then validate `:name` is always present:

```elixir
def changeset(struct, params) do
  struct
  |> cast(params, [:name])
  |> validate_required([:name])
end
```

If we use our new function and try passing the same `%{"name" => ""}` as before (or an altogether empty map), we should get an invalid changeset:

```elixir
iex> Person.changeset(%Person{}, %{"name" => ""})
#Ecto.Changeset<
  action: nil,
  changes: %{},
  errors: [name: {"can't be blank", [validation: :required]}],
  data: #Person<>,
  valid?: false
>
```

As we expected `:valid?` is no longer `true` but there's a few other things to note: `:changes` remains blank and `:errors` is now populated with a helpful error message.

If we attempt to persist this data using `Repo.insert/2` with the changeset above we'll receive `{:error, changeset}` back, this is advantagous as we do not need to check `changeset.valid?` ourselves.

### `validate_length/3`

In addition to `validate_required/2`, Ecto also provides us with `validate_length/3`.
With `validate_length/3` we can ensure a string is an exact length or whether it falls between a min and max, for all of the available options check out the [`validate_length/3`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_length/3) documentation.

For our example we'll insist that `:name` be a minimum of two characters in length which we can do using the `:min` option:

```elixir
def changeset(struct, params) do
  struct
  |> cast(params, [:name])
  |> validate_required([:name])
  |> validate_length(:name, min: 2)
end
```

If we jump into `iex` we can give our new validation a try:

```elixir
iex> Person.changeset(%Person{}, %{"name" => "A"})
#Ecto.Changeset<
  action: nil,
  changes: %{name: "A"},
  errors: [
    name: {"should be at least %{count} character(s)",
     [count: 2, validation: :length, min: 2]}
  ],
  data: #Person<>,
  valid?: false
>
```

You may be surprised that the error message contains the cryptic `%{count}` — this is to aid translation to other languages; if you want to display the errors to the person directly, you can make them human readable using [`traverse_errors/2`](https://hexdocs.pm/ecto/Ecto.Changeset.html#traverse_errors/2).

### `validate_number/3`

As we can guess `validate_number/3` validates that our value is a number and separately, if it falls within a optional range.
For our example we just need to validate they're older than 0 and within a reasonable limit, let's say 120.
Much like `validate_length/3` there are number of options available to us which can found in the [documentation](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_number/3), but for our needs we'll focus on `:greater_than` and `:less_than`.

We need to be sure to add `:age` to our `cast/4` so the value will be permitted through and we might as well add it to `validate_required/3` since it is a required field:

```elixir
def changeset(struct, params) do
  struct
  |> cast(params, [:age, :name])
  |> validate_required([:age, :name])
  |> validate_length(:name, min: 2)
  |> validate_number(:age, greater_than: 0, less_than: 120)
end
```

We won't worry about testing our `changeset/2` function here since we know already how it will function.

Some of the other built-in validators in `Ecto.Changeset` are:

+ `validate_acceptance/3`
+ `validate_change/4`
+ `validate_confirmation/3`
+ `validate_exclusion/4`
+ `validate_inclusion/4`
+ `validate_format/4`
+ `validate_subset/4`

You can find the full list with details on how ot use to use them in the documentation [summary](https://hexdocs.pm/ecto/Ecto.Changeset.html#summary).

### Custom validations

Although the built-in validators cover a wide range of use cases there are often times the need for something specific to our problem.
Lucky for us creating custom validations in Ecto is a breeze.
Since each previous `validate_` function we used accepted and returned a `%Ecto.Changeset{}`, we just need to do the same.

Let's add a custom validator to ensure `:name` isn't in a list of prohibited words.
For our example we'll keep our blacklist short (and not actually offensive):

```elixir
@prohibited_words ["bad", "offensive", "suggestive"]

def validate_name_appropriate(changeset) do
  name_unsafe =
    changeset
    |> get_field(:name)
    |> String.downcase()
    |> String.split(" ", trim: true)
    |> Enum.any?(name_parts, &(&1 in @prohibited_words))

  if name_unsafe do
    add_error(changeset, :name, "contains prohibited words")
  else
    changeset
  end
end

```

Above we introduced two new functions: [`get_field/3`](https://hexdocs.pm/ecto/Ecto.Changeset.html#get_field/3) and [`add_error/4`](https://hexdocs.pm/ecto/Ecto.Changeset.html#add_error/4).
While we can probably guess what they're doing, let's walk through our validation anyways:

+ First we use `get_field/3` to retreive the value for `:name`.
It's important to know that `get_field/3` will try to get the value from the changes first before falling back to the original data.
+ Next we downcase our string to make it easier to compare with our prohibited words.
+ We split it into parts (`"test user"` becomes `["test", "user"]`).
+ With `Enum.any?/2` we check if any part of our name is included in the prohibited words list.
+ Finally, if there we do have an unsafe name we rely on `add_error/3` to add a new error to our change.
Here we pass in our changeset, the field to which the occur refers, and finally our error message.

It is a good practice to always return a `%Ecto.Changeset{}` so we can use it in our `changeset/2` pipeline:

```elixir
def changeset(struct, params) do
  struct
  |> cast(params, [:age, :name])
  |> validate_required([:age, :name])
  |> validate_length(:name, min: 2)
  |> validate_number(:age, greater_than: 0, less_than: 120)
  |> validate_name_appropriate()
end
```

Now we can give it a try:

```elixir
iex> Person.changeset(%Person{}, %{"name" => "Offensive Name"})
#Ecto.Changeset<
  action: nil,
  changes: %{},
  errors: [name: {"contains prohibited words", []}],
  data: #Person<>,
  valid?: false
>
```

No surprise to us, our new validation rule works!

## Adding changes programatically

Sometimes you want to introduce changes to a changeset manually.
We previously saw how to use `changes/2` to apply a map of changes to our changeset, now we'll look at a different approach using `put_change/3`.

To demonstrate `put_change/3` we'll update our `changeset/2` function to allow `:name` to be blank and when it's blank set it to `"Anonymous"`.
In order to accomplish this let us create a new `set_anonymous/1` function that accepts and returns a changeset as we did earlier:

```elixir
def set_anonymous(changeset) do
  name = get_field(changeset, :name)

  if is_nil(name) or name == "" do
    put_change(changeset, :name, "Anonymous")
  else
    changeset
  end
end
```

Nothing to difficult, right?  If the `:name` is blank or an empty string we want to put a change on to the changeset making `:name` equal to `"Anonymous"`.

With our new function in place, let's update see what `changeset/2` looks like now:

```elixir
def changeset(struct, params) do
  struct
  |> cast(params, [:age, :name])
  |> validate_required([:age])
  |> validate_number(:age, greater_than: 0, less_than: 120)
  |> validate_name_appropriate()
  |> set_anonymous()
end
```

Now if we don't pass `:name` then it will be set to `"Anonymous"` as expected:

```elixir
iex> Person.changeset(%Person{}, %{"age" => 99})
#Ecto.Changeset<
  action: nil,
  changes: %{age: 99, name: "Anonymous"},
  errors: [],
  data: #Person<>,
  valid?: true
>
```

## Use-case specific changesets

A single `changeset/2` function capturing all of the rules of our example application makes sense but for something larger it almost certainly does not.
Having changeset functions with specific responsibilities is not uncommon.

In an application with account registration you may have `registration_changeset/2` and later use `update_changeset/3` to apply updates to an existing user.

## Schemaless changesets

An exciting feature of `Ecto.Changeset` is the ability to apply changesets to more than just our Ecto schemas.
With schemaless changes we can provide the same level of data validation and transformation, without the schema.

While they are beyond the scope of these lessons, everything we need to know can be found in the official [schemaless changesets](https://hexdocs.pm/ecto/Ecto.Changeset.html#module-schemaless-changesets) docs.

## Conclusion

This is just the tip of the iceberg when it comes to the punch that `Ecto.Changeset`'s packs.
As the following lessons build on what we've covered here today, we'll explore more advanced usages of changeset to include working with associated data.
