---
title: Valorant Authentication & Endpoints
created: 4/23/23
---

Videos games are cool. I think most of us can agree on that. I've recently been really into a game called [VALORANT](playvalorant.com/), by [Riot Games](www.riotgames.com/).  After playing for a while, I wanted to build something with their API. I was disappointed to find out that it was pretty old, and the only way to interact with matches was to have a registered application, and there was no way my destined to fail side project would be considered. I eventually stumbled upon https://valapidocs.techchrism.me, which details the internals of Valorant's production servers. I was fascinated, and made the library [ValClient.js](https://github.com/darkdarcool/valclient.js) to make it easier for other developers to create their own side projects with Valorant. Eventually, I found a custom Valorant client called [Assist](https://github.com/HeyM1ke/Assist), and I loved it. Some of the things it did I had never seen before, like authenticating with Riot Client, instead of just with username and password. I decided to create my own custom client, and I named it DarkClient, which is still a work in progress! Anyway, I've decided to talk about how making a custom client works, since it was a complicated process so far.

 # Getting Started
 Any programming language will suffice, I myself use Electron + NextJS, and Assist uses C# + Alvonia, but you can really use anything you want, as long as you have access to the users local machine.

# Authentication
Username & password auth is fairly simple, first, you need to make a `POST` request to `https://auth.riotgames.com/api/v1/authorization`, and store the cookies from the request. I myself do it like this in JS/TS: 
```ts
await axios.post("https://auth.riotgames.com/api/v1/authorization", JSON.stringify({
      "client_id": "play-valorant-web-prod",
      "nonce": "1",
      "redirect_uri": "https://playvalorant.com/opt_in",
      "response_type": "token id_token",
      }), {
        jar: cookieJar, // from the package 'tough-cookie'
        headers: {
          "Content-Type": "application/json",
          "User-Agent": "riotgames"
        },
        withCredentials: true,
});
```
With the required cookies now stored, you can now make the official auth request! To do this, you need to make a `PUT` request to `https://auth.riotgames.com/api/v1/authorization`,  and attach the `username` and `password` of the user you're trying to log in to the body of the request. I do this myself like this:
```ts
let data = await axios.put("https://auth.riotgames.com/api/v1/authorization", JSON.stringify({
      "type": "auth",
      "username": "USERNAME", // username of the user
      "password": "PASSWORD", // password of the user
      "remember": true // true/false
      }), {
        jar: cookieJar,
        headers: {
          "Content-Type": "application/json",
          "User-Agent": "riotgames"
        },
        withCredentials: true
    });
```
NOTE: You may need to do a [Multi Factor Request](https://valapidocs.techchrism.me/endpoint/multi-factor-authentication) in case the user has that enabled.

You can then find the token for the user in the `uri` property of the request. 

## Riot Client Auth
This one was a doozy, and took me a while to iron out all the kinks, but I think it should work (most of the time) now! 

First, you need the user to sign out of Riot Client, and this can be done by deleting the file `C:\Users\<USER>\AppData\Local\Riot Games\Riot Client\Data\RiotGamesPrivateSettings.yaml`. After opening Riot Client and the user logs in (make sure they enabled `Remember Me`!), you can read the yaml file, and in the cookies property, get the `ssid` cookie, which is the only one you need. 

Then, you make a regular old cookie auth request, but you attach the `ssid` cookie, and I do it like this:
```ts
const resp = await fetch('https://auth.riotgames.com/authorize?redirect_uri=https%3A%2F%2Fplayvalorant.com%2Fopt_in&client_id=play-valorant-web-prod&response_type=token%20id_token&nonce=1', {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json',
      "Access-Control-Allow-Origin": "*",
      "Cookie": `ssid=${tokens.ssid};`
    },
    follow: 1,
});
```

Why the follow property? It redirects you to a Valorant page, which isn't needed. You only need the first page it sends you to. 

You can then get the access token by using this script on the response:

```ts
 let url = resp.url;
 let accessToken = url.split("#access_token=")[1].split("&")[0];
```

## How to get the Entitlement Token

This is pretty much required for all useful endpoints. When you have the access_token, you can get the token like this:
```ts
let eTokenReq = await fetch("https://entitlements.auth.riotgames.com/api/token/v1", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${accessToken}`,
      "Content-Type": "application/json"
    }
  });
```
You can then find the `entitlements_token` in the body of the response. 

## Get user info

It's pretty simple, and you only need to access_token for this one:

```ts
let userInfoReq = await fetch("https://auth.riotgames.com/userinfo", {
	headers: {
		"Authorization": `Bearer ${accessToken}`
	}
});
```
The response is detailed [here](https://valapidocs.techchrism.me/endpoint/player-info).

## Any other weird auth methods?

There is one other way, which is using Valorant's local API, which has an endpoint for getting its tokens. The local API on its own is a post in its own right, but for now, you can find information about it [here](https://valapidocs.techchrism.me/endpoint/entitlements-token). 

## How do I use the other endpoints now?

First, you need to prepare the headers for each request, you need: `Authorization`, `X-Riot-Entitlements-JWT`, `X-Riot-ClientPlatform`, and sometimes `X-Riot-ClientVersion`.

If you followed the authorization steps, you already have the `Authorization` & `X-Riot-Entitlements-JWT` headers:

```ts
const headers = {
	"Authorization": `Bearer ${accessToken}`,
	"X-Riot-Entitlements-JWT": ${entitlements_token},
	"X-Riot-ClientPlatform": "ew0KCSJwbGF0Zm9ybVR5cGUiOiAiUEMiLA0KCSJwbGF0Zm9ybU9TIjog" + 
    "IldpbmRvd3MiLA0KCSJwbGF0Zm9ybU9TVmVyc2lvbiI6ICIxMC4wLjE5" + 
    "MDQyLjEuMjU2LjY0Yml0IiwNCgkicGxhdGZvcm1DaGlwc2V0IjogIlVua25vd24iDQp9",
    "X-Riot-ClientVersion": "..." // The version of valorant that you're using, like `release-02.00-shipping-4-0-0`
}
```
With the headers, you now need to get the base urls for the requests. These are the `glz` and `pd` urls. If you're running the script on your users machine that already has Valorant installed and it's been run before, you can use this script:

```ts
export type Urls = {
  pd_url: string,
  glz_url: string,
  version: string,
}

export function getUrls(): Urls {
  let path = process.env["LOCALAPPDATA"] + '\\VALORANT\\Saved\\Logs\\ShooterGame.log';
  let region = "na";
  let glz: string[] = [];
  let version = "release-02.00-shipping-4-0-0";
  readFileSync(path, 'utf-8').split(/\r?\n/).forEach(function(line) {
    if (line.includes(".a.pvp.net/account-xp/v1/")) {
      region = line.split('.a.pvp.net/account-xp/v1/')[0].split(".").slice(-1)[0];
    }
    else if (line.includes('https://glz')) {
      glz = [(line.split('https://glz-')[1].split(".")[0]), (line.split('https://glz-')[1].split(".")[1])];
    }
    else if (line.includes("CI server version:")) {
      version = line.split("CI server version:")[1].trim(); 
    }
  });
  if (region == "pbe") { 
    region = "na";
    glz = ["na1", "na"]
  }
  let pd_url = `https://pd.${region}.a.pvp.net`;
  let glz_url = `https://glz-${glz[0]}.${glz[1]}.a.pvp.net`
  return { pd_url, glz_url, version } as Urls;
}
```
You can also find a Python version of the script [here](https://github.com/zayKenyon/VALORANT-rank-yoinker/blob/ac2002fc75b4716b6138a7846e62621fedcb598e/src/requestsV.py#L175).

Now that you have these urls, you can now use the endpoints from https://valapidocs.techchrism.me to your hearts content!

# Conclusion

That's pretty much all she wrote for now. I'll update this more and more because there's still a lot to share (like local client, and XMPP), but we'll save that for another time!

This is darkdarcool, signing off!
