# Horizen CRM

A CRM I built for my own web studio and use every day to run the sales pipeline and client delivery. The source is private because it holds real client data — this repo is the architecture behind it, with the two parts worth reading pulled out.

Stack: React 19, TanStack Start, Supabase (Postgres, RLS, Edge Functions on Deno), OpenAI.

## How leads get in

The public site's contact form doesn't write to the CRM tables directly. It calls one Postgres function, `submit_website_inquiry`, using the site's anon key:

```sql
create or replace function submit_website_inquiry(p_name text, p_contact text, p_message text)
returns uuid language plpgsql security definer set search_path = public as $fn$
declare
  v_client_id uuid;
begin
  select id into v_client_id from clients where lower(email) = lower(p_contact) limit 1;
  if v_client_id is null then
    insert into clients (name, email, source, notes)
    values (p_name, p_contact, 'website', p_message)
    returning id into v_client_id;
  end if;
  insert into deals (client_id, stage, title, notes)
  values (v_client_id, 'discovery', 'Website inquiry', p_message);
  return v_client_id;
end; $fn$;
```

`security definer` means the function runs with the owner's privileges, so it can insert into `clients` and `deals` even though the `anon` role calling it has zero read or write access to either table under RLS. The alternative — granting `anon` insert rights on those tables directly — means trusting every future schema change to keep the intake path narrow. One function I can read top to bottom felt safer than trusting policies to stay tight forever.

This used to run off a booking widget instead: inserting into a `bookings` table fired an `AFTER INSERT` trigger that did the same client-and-deal creation. I killed it once the data showed nobody was using the booking flow — one path to maintain beats two, even though the trigger version arguably matched "this should happen no matter how the row got inserted" more cleanly than a direct RPC call does.

## The AI feature doesn't trust the model

The CRM has one AI feature: drafting first-contact outreach messages from a Supabase Edge Function. The interesting part isn't the OpenAI call — it's that the function doesn't trust what comes back. The model is told never to use em-dashes and to reference a portfolio link with a `[link]` placeholder, and the function enforces both anyway, because a prompt instruction is a preference, not a guarantee:

```ts
function stripDashes(text) {
  return typeof text === "string" ? text.replace(/[—–]/g, ",") : text;
}

function insertLink(text) {
  return typeof text === "string" ? text.replaceAll("[link]", PORTFOLIO_URL) : text;
}
```

If the model ignores the instruction anyway, the output is still correct. A hallucinated or mistyped link can never reach a lead, because the real URL was never something the model got to write.

## What's not great

The schema lives as a SQL script I paste into Supabase's editor, not as tracked migrations, so there's no real history of how it evolved and no generated TypeScript types — nothing catches a renamed column until it 500s at runtime. It's also built for one user. There's no role separation between me and anyone else who might eventually touch this.

---

Matija Radulović · [horizen.rs](https://horizen.rs)
