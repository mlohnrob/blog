---
layout: post
title: "Hvordan Jeg Laver Session Authentication"
date: 2023-03-10 12:00:00 +0200
category: "hvordan-jeg"
tags: ["session", "software-arkitektur", "postgres", "express", "node.js", "http"]
---

# Indledning
Authentication er en vigtig del af næsten alle større applikationer, og det kan laves på flere forskellige måder. Den
simpleste måde er at bare sende brugernavn og adgangskode med i hvert request til backend-serveren, så den kan
autentificere hver gang - dette er en dårlig løsning, fordi det bringer klientens bruger i fare for ting som Man In the
Middle angreb og det bruger unødvendigt mange ressourcer på serveren. En populær metode til authentication er brugen af
JWT. JWT (JSON Web Token) er et JSON-objekt, som beskriver brugeren og hvilke rettigheder den har. Denne token er
kryptisk signeret af serveren, så den kan valideres. Dog har JWT nogle ulemper: Fordi udløbstidspunktet ligger i token'en,
så er det svært at tilbagekalde token'ens validitet. Alle brugerens rettigheder bliver sendt til serveren i hvert
request, og disse JWT kan godt gå hen at blive ret store; derfor bliver der brugt unødvendigt meget bredbånd. Af disse
grunde er jeg fan af brugen af session ID til authentication.

# Session ID
Jeg designer min session ID app-arkitektur således:

## Klienten logger ind med email og password.
I frontenden er der en login-side, hvor man som bruger kan taste sin email og adgangskode. Når brugeren trykker "Login",
så sendes et POST request til serveren med email og password som headers.
```HTTP
POST /login/ HTTP/1.1
Host: api.example.com
Email: august@emmery.dk
Password: 12345
```

## Serveren modtager login request og laver session i database
Når serveren modtager et login POST request, så sender den email og password videre til databasen for at blive tjekket.
```javascript
app.post("/login/", async (req, res) {
  const { email, password } = req.headers;
  if (email == null || password == null) res.status(400).send("Fields not populated.");

  try {
    // Email og password sendes til databasen
    const dbResponse = await dbClient.query("SELECT * FROM fn_user_login($1, $2);", [email, password]); 
    const sessionId = dbResponse.rows[0]["fn_user_login"];

    // Session ID sendes tilbage til klienten
    res.status(201).send(sessionId);
  } catch (e) {
    console.error(e);
    res.status(500).send("Internal Server Error");
  }
});
```

I databasen er der en tabel, sessions, som er defineret således:
```sql
CREATE TABLE sessions(
  session_id UUID NOT NULL UNIQUE PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES public.users(id),
  expiration TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP + interval '1 week', --Dette interval kan selvfølgelig ændres
);
```

Og denne funktion bliver kaldt af API-serveren med klientens email og password:
```sql
CREATE FUNCTION fn_user_login(
  email TEXT,
  password TEXT
) RETURNS UUID
  LANGUAGE 'plpgsql'
AS $$
DECLARE sessionId;
BEGIN
  IF ((SELECT users.password FROM public.users WHERE users.email = $1) != $2) THEN
    RAISE EXCEPTION 'Email and password combination does not exist';
  END IF;

  INSERT INTO public.sessions(user_id) VALUES
  SELECT users.id FROM public.users WHERE users.email = $1
  RETURNING sessions.id INTO sessionId;

  RETURN sessionId;
END;
$$
```

Session ID'et kunne f.eks. se sådan ud: 10f0109e-01e8-4b55-9eb6-e954a45e2d5c

## Klienten gemmer session_id i en cookie eller anden local storage.
Når klienten modtager dette session ID i svaret fra serveren, så gemmes det som en cookie i browseren. Dette er smart,
fordi klienten så kan bruge dette session ID, som er gemt som cookie, i fremtidige requests til serveren uden behovet
for at sende email og password med.

## Fremtidige requests
F.eks. kunne et fremtidigt request se således ud:
```http
GET /items/ HTTP/1.1
Host: api.example.com
SessionId: 10f0109e-01e8-4b55-9eb6-e954a45e2d5c
```
Her modtager serveren altså kun et session ID; ikke en email og adgangskode. Serveren kan så bruge dette session ID, 
hvis det ikke er udløbet, til at regne ud hvilken bruger, der er tilknyttet det, og dermed hvilke rettigheder der er tilknyttet.

På denne måde kan serveren altid tilbagekalde en sessions validitet ved blot at ændre udløbstidspunktet af session ID'et
inde i sessions-tabellen.

# Konklusion
Session ID-metoden er en god metode til at håndtere authentication på serveren. Selvom disse eksempler selvfølgelig er
simplificerede for din skyld, så er de ikke langt fra, hvad man bruger i produktion; koncepterne, som er det vigtigste,
er de samme.
