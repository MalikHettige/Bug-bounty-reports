05/06/2026

# How I Would Have Found This: Access to Private Photos of Apps in App section (IDOR)

## The Target

Shopify is an e-commerce platform where developers can create and sell apps.
The "App section" is where developers manage their apps, including uploading photos.
The bug is in the photo viewing feature.

## My Recon Approach

I would start by browsing the Shopify App Store as a developer.
I would upload an app and look at all the URLs that load my app's data.
I would pay attention to any URL with a number in it — like `?id=12345` or `/photo/12345`.
Numbers in URLs often mean IDOR.

## The Vulnerability Class

IDOR — Insecure Direct Object Reference.
The application uses a number (the photo ID) to decide which photo to show.
But it does not check if the logged-in user actually owns that photo.
So changing the number shows someone else's photo.

## My Testing Steps

1. Create a Shopify developer account.
2. Upload an app with a photo.
3. Go to the app section and find the URL that shows the photo.
4. Note the photo ID in the URL (e.g., `id=12345`).
5. Log out. Create a second developer account. Upload a different app with a different photo.
6. Note the second photo ID (e.g., `id=12346`).
7. Log in as the first user. Change the URL from `id=12345` to `id=12346`.
8. If I see the second user's private photo, I have found an IDOR.

## Why I Would Have Caught It

I know that numbers in URLs are dangerous.
Anytime I see `?id=`, `?user_id=`, `?photo_id=`, I test changing the number.
This is a basic IDOR test that takes 30 seconds.
Most hunters skip it because they think "surely they check ownership."
But many apps don't.

## What I Learned for My Hunting

- IDOR is everywhere. Even big companies like Shopify miss it.
- One parameter change can be a valid bug.
- The impact is privacy breach — seeing someone else's private photos.
- Report writing matters for severity. This was downgraded from High to Low.
Better impact explanation might have kept it High.
- I will test every numbered parameter I find in every app I hunt.
