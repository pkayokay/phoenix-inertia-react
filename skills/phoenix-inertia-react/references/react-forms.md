# React + TypeScript Forms

## Prefer `<Form>`

Collects fields from `name` attributes. No manual form state for simple CRUD.

Phoenix expects nested params (`user[name]`). Validation error keys come from the
**changeset**, so a `User` `:name` error is `errors.name` (string key `"name"`),
not `errors["user.name"]`. Use dotted keys only for nested changeset embeds/
assocs (`"team.name"`, `"items[1].price"`).

```tsx
import { Form } from "@inertiajs/react";

type Props = {
  user?: { id: number; name: string; email: string };
};

export default function UserForm({ user }: Props) {
  return (
    <Form
      action={user ? `/users/${user.id}` : "/users"}
      method={user ? "put" : "post"}
    >
      {({ errors, processing, progress }) => (
        <>
          <label>
            Name
            <input type="text" name="user[name]" defaultValue={user?.name ?? ""} />
          </label>
          {errors.name && <p>{errors.name}</p>}

          <label>
            Email
            <input type="email" name="user[email]" defaultValue={user?.email ?? ""} />
          </label>
          {errors.email && <p>{errors.email}</p>}

          <label>
            Avatar
            <input type="file" name="user[avatar]" />
          </label>
          {progress && <progress value={progress.percentage} max={100} />}

          <button type="submit" disabled={processing}>
            Save
          </button>
        </>
      )}
    </Form>
  );
}
```

Edit forms: `defaultValue={…}` — avoid controlled `value=` unless you intentionally
manage state (controlled inputs break `<Form>` dirty tracking).

## When to use `useForm`

Multi-step wizards, dynamic add/remove fields, or form data shared with siblings:

```tsx
import { useForm } from "@inertiajs/react";

type UserForm = {
  user: {
    name: string;
    email: string;
  };
};

export default function New() {
  const { data, setData, post, processing, errors } = useForm<UserForm>({
    user: { name: "", email: "" },
  });

  function submit(e: React.FormEvent) {
    e.preventDefault();
    post("/users");
  }

  return (
    <form onSubmit={submit}>
      <input
        value={data.user.name}
        onChange={(e) => setData("user", { ...data.user, name: e.target.value })}
      />
      {errors.name && <p>{errors.name}</p>}
      <button type="submit" disabled={processing}>
        Create
      </button>
    </form>
  );
}
```

Nest `data` the way Phoenix controllers expect (`%{"user" => …}`). Error keys
still follow the changeset (`name`), not the nest path.

## NEVER

- `react-hook-form` / Formik — fights Inertia CSRF, redirects, error mapping
- Parallel `axios.post` for the same mutations Inertia should own
- Flat `useForm({ name, email })` when the controller pattern-matches `%{"user" => params}`
- Mapping `errors["user.name"]` for a top-level `:name` changeset error

## Server pairing

```elixir
# success
conn |> put_flash(:info, "Saved") |> redirect(to: ~p"/users")

# failure — PRG with errors
conn |> assign_errors(changeset) |> redirect(to: ~p"/users/new")
```

## CSRF

Adapter sets `XSRF-TOKEN` cookie. Client must send `x-csrf-token`:

```ts
axios.defaults.xsrfHeaderName = "x-csrf-token";
```

Keep this in `app.tsx`. `<Form>` / `useForm` use the Inertia client.

## Files

Use `<input type="file" />` inside `<Form>` / `useForm`. Inertia switches to
`FormData` automatically. Ensure the endpoint/changeset accepts the upload.

## Delete

```tsx
import { router } from "@inertiajs/react";

<button
  type="button"
  onClick={() => {
    if (confirm("Delete?")) router.delete(`/users/${id}`);
  }}
>
  Delete
</button>
```

Or `<Form method="delete" action={`/users/${id}`}>`.
