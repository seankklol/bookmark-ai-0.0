# Bookmark AI 0.0

## Repo Set Up

1. clone https://reactstarter.xyz/
2. connect to Github
3. ```npm install --legal-peer-deps```
4. ```npm run dev```
5. ```npx convex dev```
6. Goto .env.example and copy all
7. Paste that in .env.local (keep keys already in .env.local)
8. set NEXT_PUBLIC_CONVEX_URL the same as VITE_CONVEX_URL
9. Log into https://clerk.com/ 
10. Create a new project
11. Goto 'Configure' then 'API Keys'
12. Select 'React Router' in 'Quick Copy' and Copy that
13. Paste that in .env.local
15. Paste 'http://localhost:####' into .env.local without the '/' in the end
16. Goto Configure -> JWT Templates -> Choose Convex Template -> Copy the 'Issuer'
17. Goto Convex -> Settings -> Enviornment Variables. Paste value you copied to 'VITE_CLERK_PROJECT_FRONTEND_API_URL'
18. Goto https://sandbox.polar.sh/
19. Create new orgnaization
20. Add the pricing
21. Goto Dashboard -> Settings -> Scroll to Developers -> New Token -> Select All -> Select Expiration Date -> Name it something (idk) -> Click Create -> Copy Value
22. Goto Convex -> Settings -> Enviornment Variables. Paste value you copid to 'POLAR_ACCESS_TOKEN'
23. Goto Polar Settings and copy the ID. Paste that in Convex Enviornment Variables as 'POLAR_ORGANIZATION_ID'
24. Goto Convex -> Settings -> URL & Deploy Key -> Show Development Credentials -> Copy HTTP Actions URL
25. Goto Polar -> Settings -> Webhook -> Add Endpoint -> Paste value, but dont save yet.
26. Goto Repo -> convex/http.ts -> Scroll towards the end and you'll find '/payments/webhook' as path
27. Go back to polar and add '/payments/webhook' after the HTTP Actions URL you pasted earlier. Make sure there are no double slashes
28. Format Raw and copy everything under events. Click create then copy the secret
29. Goto Convex and paste value in enviornment variables as 'POLAR_WEBHOOK_SECRET'
30. Get OpenAI API key ans paste that in Convex Enviornment Variables
31. install ngrok if needed. After that, run ```ngrok http ####``` fill in #### with your localhost numbers
32. goto the ngrok website displayed in terminal. 
33. Copy some value like '21f574b417f1.ngrok-free.app' and paste that into vite.config.js. It should look smth like this:
```
export default defineConfig({
  plugins: [tailwindcss(), reactRouter(), tsconfigPaths()],
  server: {
    allowedHosts: [
      "21f574b417f1.ngrok-free.app"
    ],
  }
});
```
34. Copy the ngrok website URL and paste that into Convex Enviornment Variable as 'FRONTEND_URL'
35. You're Done Settting this project up!

## Project Set Up (When you close and reopen ngrok)

Next time you reopen this project, repeat steps #31~34 again.
Run: ```npm run dev```, ```npx convex dev```, ```ngrok http ####```
Open: Convex, Clerk, Polar(sandbox.polar.sh -- sandbox is for pre-production), localhost(faster)/ngrok(slow)
Remember to git commit!!!