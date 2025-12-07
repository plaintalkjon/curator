Plan – Curator_Van_Gogh Bot (JavaScript + Supabase)1. Goal & Success Criteria

Goal: Post two random public‑domain Van Gogh artworks to a Mastodon account (mastodon.social) every day.
Success:

Two posts appear on the bot timeline each day (approximately 09:00 UTC and 21:00 UTC).
Images are correctly attributed (artist, title, year, source URL).
No uncaught exceptions; logs contain “✅ posted”.
All secrets (Mastodon token, API keys) stored securely in Supabase.




2. Tech Stack
LayerTechnologyReasonLanguageNode.js (v20+)You already know JavaScript; async/await works well with HTTP APIs.Artwork sourceThe MET Collection API (open‑access Van Gogh works)Free, public‑domain images with reliable URLs and metadata.Database & SecretsSupabase (Postgres + Edge Functions)Stores Mastodon token, schedule, and posting history.SchedulerSupabase Scheduled Functions (cron)Native serverless cron, zero extra infra.Mastodon clientmastodon-api NPM packageThin wrapper for media upload and status creation.HostingSupabase Edge Functions (or local supabase functions serve)TLS, logs, and automatic scaling.

3. Supabase Schema
-- Bot configuration (single row for this bot)
create table bots (
  id uuid primary key default gen_random_uuid(),
  mastodon_instance text not null,           -- e.g. https://mastodon.social
  access_token text not null,                -- encrypted via RLS
  content_warning text default null,         -- optional CW
  name text not null default 'Curator_Van_Gogh'
);

-- Schedule: two daily posts
create table schedule (
  bot_id uuid primary key references bots(id),
  cron_expr text not null                    -- e.g. '0 9,21 * * *' (09:00 & 21:00 UTC)
);

-- History of posted artworks
create table posts (
  id uuid primary key default gen_random_uuid(),
  bot_id uuid references bots(id),
  met_object_id integer not null,            -- MET object ID for traceability
  image_url text not null,
  title text,
  year text,
  source_url text,
  posted_at timestamptz not null default now(),
  status_id text not null                    -- Mastodon post ID
);
Enable Row‑Level Security on bots.access_token so only the service role can read it.

4. Workflow (Supabase Edge Function)

Cron trigger fires according to schedule.cron_expr (0 9,21 * * *).
Function reads the single row from bots.
Fetch a random Van Gogh work from the MET API:

Query https://collectionapi.metmuseum.org/public/collection/v1/search?artistOrCulture=Vincent%20van%20Gogh&hasImages=true once to get the full list of IDs (store this list in Supabase cache or memory).
Pick a random ID from that list.
Retrieve object details via .../objects/<ID>.
Extract primaryImageSmall (or primaryImage) URL, title, objectDate, and objectURL.


Download the image (binary buffer).
Upload image to Mastodon (mastodon.mediaUpload).
Compose status text (example):
🎨 Van Gogh – "Starry Night" (1889)
Source: https://www.metmuseum.org/art/collection/search/437980
#VanGogh #Art

Include content_warning if desired.
Create Mastodon status with the uploaded media_id.
Insert a row into posts with all metadata (MET ID, URLs, timestamps, Mastodon status_id).
Log ✅ Curator_Van_Gogh posted <title>.
On any error, catch, log ❌ Curator_Van_Gogh failed: <error>, optionally insert into a failed_posts table for later retry.


5. Implementation Sketch (TypeScript for Supabase Edge Function)
// src/index.ts  – Supabase Edge Function entry point
import { createClient } from '@supabase/supabase-js';
import Mastodon from 'mastodon-api';
import fetch from 'node-fetch';

// Supabase client (environment variables set in function config)
const supabase = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_ANON_KEY')!
);

// ---------- Helper: get random Van Gogh MET object ----------
async function getRandomVanGogh(): Promise<any> {
  // Cache the full list of IDs in Supabase (optional) – for simplicity we fetch each time
  const searchRes = await fetch(
    'https://collectionapi.metmuseum.org/public/collection/v1/search?artistOrCulture=Vincent%20van%20Gogh&hasImages=true'
  );
  const searchData = await searchRes.json(); // { total: X, objectIDs: [...] }
  const ids = searchData.objectIDs;
  const randomId = ids[Math.floor(Math.random() * ids.length)];

  const objRes = await fetch(
    `https://collectionapi.metmuseum.org/public/collection/v1/objects/${randomId}`
  );
  const obj = await objRes.json(); // contains title, objectDate, primaryImage, objectURL, etc.
  return obj;
}

// ---------- Helper: post to Mastodon ----------
async function postToMastodon(
  bot: any,
  imageBuffer: Uint8Array,
  statusText: string
): Promise<string> {
  const M = new Mastodon({
    access_token: bot.access_token,
    api_url: `${bot.mastodon_instance}/api/v1/`,
  });

  const mediaResp = await M.post('media', {
    file: Buffer.from(imageBuffer),
    description: statusText,
  });

  const statusResp = await M.post('statuses', {
    status: statusText,
    media_ids: [mediaResp.id],
    spoiler_text: bot.content_warning ?? '',
  });

  return statusResp.id;
}

// ---------- Main handler ----------
export default async function handler(_req: Request) {
  // 1️⃣ Load bot config
  const { data: bot, error } = await supabase
    .from('bots')
    .select('*')
    .single();

  if (error) throw error;

  try {
    // 2️⃣ Get random artwork
    const art = await getRandomVanGogh();

    // Guard: ensure image exists
    const imgUrl = art.primaryImageSmall || art.primaryImage;
    if (!imgUrl) throw new Error('Selected object has no image');

    const imgBuf = await fetch(imgUrl).then((r) => r.arrayBuffer());

    // 3️⃣ Build status text
    const title = art.title || 'Untitled';
    const year = art.objectDate || 'unknown year';
    const source = art.objectURL;
    const statusText = `🎨 Van Gogh – "${title}" (${year})\nSource: ${source}\n#VanGogh #Art`;

    // 4️⃣ Post to Mastodon
    const statusId = await postToMastodon(bot, new Uint8Array(imgBuf), statusText);

    // 5️⃣ Record the post
    await supabase.from('posts').insert({
      bot_id: bot.id,
      met_object_id: art.objectID,
      image_url: imgUrl,
      title,
      year,
      source_url: source,
      status_id: statusId,
    });

    console.log(`✅ Curator_Van_Gogh posted "${title}"`);
  } catch (e) {
    console.error(`❌ Curator_Van_Gogh failed:`, e);
    // optional: await supabase.from('failed_posts').insert({ bot_id: bot.id, error: e.message });
  }

  return new Response('OK', { status: 200 });
}
No external AI model is required; the MET API supplies public‑domain images.

6. Deployment Steps

Create a Supabase project and enable Edge Functions.
Run the SQL schema above in the Supabase SQL editor.
Insert a bot row with:

mastodon_instance = 'https://mastodon.social'
access_token = '<your_bot_access_token>' (keep it secret).
Optional content_warning if you want a CW.


Add a schedule row for the same bot_id with cron_expr = '0 9,21 * * *'.
Create a new Edge Function named curator_van_gogh and paste the TypeScript code.
Set environment variables for the function:

SUPABASE_URL
SUPABASE_ANON_KEY
(no extra API keys needed for MET).


Deploy: supabase functions deploy curator_van_gogh.
Monitor via Supabase Logs – look for ✅ Curator_Van_Gogh posted ….


7. Risks & Mitigations
RiskMitigationMET API downtime or rate limitingCache the full list of Van Gogh IDs in a Supabase table (van_gogh_ids) refreshed daily; fallback to cached list if the API call fails.Duplicate posting (same artwork)Store the met_object_id in posts; before posting, check the last 30 entries and avoid repeats.Mastodon rate‑limit (300 posts/day)Only two posts per day; well under the limit.Missing image URLGuard against primaryImage being empty; if missing, pick another random ID.Token leakageaccess_token stored encrypted; enable RLS so only the service role can read it.Unexpected errors causing missed postsAll steps wrapped in try/catch; failures logged and can be retried manually via a simple script.

8. Future Enhancements (post‑prototype)

Multiple artists: add an artists table; schedule per‑artist posting times.
User‑submitted prompts: expose a tiny Supabase‑backed UI where followers can request a specific Van Gogh work.
Image processing: add a light watermark or resize step before upload (Sharp library).
Analytics: store engagement metrics (likes, boosts) in a metrics table for later reporting.
Multi‑instance scaling: if you ever need more than two posts per day, split the schedule across separate Edge Functions or use a queue (Supabase Realtime).


Copy the entire content above into plan.md. Cursor will parse the headings, understand the data model, and follow the step‑by‑step workflow to build the Curator_Van_Gogh bot.