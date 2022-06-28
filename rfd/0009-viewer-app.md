---
authors: Mathew (wrm932@gmail.com)
state: draft
---

# Required Approvers
* Engineering @pierrebeaucamp && @jimbishopp && @michellescripts && @mcbattirola

# RFD 0009 - RFDs

## What

Create a directory viewer similar to Github and Google drive. 


## Why

A pratical exam that tests the potential applicant on their fullstack software knowledge. 

## Details

Each RFD is stored in a markdown file under
https://github.com/gravitational/teleport/tree/master/rfd and has a unique
number.

The directory viewer should be allow the user to navigate a directory structure by clicking on files or folders. A bread crumb navigation should inform them of where they currently are and allow them to navigate back to their previously clicked directory location. The user should also be able to filter out a list of files using a search box. Search/filtering will be done client side. 

 In addition, user authentication should be implemented so only logged in users will be able to access the viewer. If a user is unauthenticated they should be redirected back to the login form. However, if given a direct url that points to a specific directory. The application should be able to remember it and redirect the unauthenticated user to the proper url after successful authentication. 

## Techstack 

We will be using Next.js for web application development. Because it is a full-stack framework (as in, it handles both the frontend and backend of your application).

It simplifies building full-stack applications using React. It provids automatic routing for pages and built-in methods for server-side rendering (SSR) and data fetching. 

## Testing Strategy

Ideally we would like to write a complete testing suite which covers unit tests for both the backend and frotend. And adding an E2E test which coveres common user scenarios. For the purpose of this exercise will be focusing on creating unit tests for the React components. 

Testing checklist
* File view table w/ sorting.
* Url based routing which shows the appropriate file directory structure. 


## API 

`POST /login`
```json
POST /login HTTP/1.1
Content-Type: application/json; charset=utf-8
Host: localhost:3000

{"username": "pikachu", "password": "Vlog7-Ancient-Outdoorsy"}
```

`POST /logout`
```json
POST /logout HTTP/1.1
Content-Type: application/json; charset=utf-8
Host: localhost:3000
```

```json
200 GET /directory/[...slug]
{
  name: "teleport",
  size: 0, // bytes
  type: "dir",
  items: [
    {
      name: "lib",
      size: 0, // bytes
      type: "dir",
    },
    {
      name: "README.md",
      size: 4340000, // bytes
      type: "file",
    },
  ],
}

```

Return codes for `/directory` endpoint
* `200` success response
* `401` unauthenticated response
* `500` server error response


### Security

Let's cover the most obvious attack vector which is the user facing side of the application. Listed below is not an exhaustive list of vulnerabilities but I'll discuss some of the more well known and common threats. 

#### 1. Cross-Site Scripting (XSS)
* XSS is a serious client-side vulnerability. A perpetrator is able to add some malicious code to to our program that is interpreted as valid and is executed as a part of the application. This compromises the functionality of the app and the user data. Luckily, React outputs elements and data inside them using auto escaping. We can further secure our app by using a library such as [DOMPurify](https://github.com/cure53/DOMPurify). 

    * Validate all data that flows into app from the server or a third-party API. This cushions the app against an XSS attack, and at times, you may be able to prevent it, as well.

    * Don't mutate DOM directly. If you need to render different content, use innerText instead of innerHTML. Be extremely cautious when using escape hatches like findDOMNode or createRef in React.

    * Always try to render data through JSX and let React handle the security concerns for you.

    * Avoid writing your own sanitization techniques. It's a separate subject on its own that requires some expertise.

    * Use good libraries for sanitizing your data. There are a number of them, but you must compare the pros and cons of each specific to your use case before going forward with one.

#### 2. Broken Authentication
* Another common issue in React.js applications is inadequate or poor authorization. This can result in attackers hacking user credentials and carrying out brute force attacks. There are various risks associated with broken authorization, like session IDs being exposed in URLs, easy and predictable login details being discovered by attackers, the unencrypted transmission of credentials, persisting valid sessions after logout, and other session-related factors.

We can implement safe session management. A lot of times developers store session-id in front-end URLs, making it clearly visible to users as well as attackers. Instead, you could store the session-id inside browser storage and use it via a custom React hook in whichever component or page of your app where you need it. We can facilitate this by creating a `useSession` hook. 

```javascript
import { useEffect, useState } from "react"

export default function useSession({session_id}){

    const [sessionId,setSessionId]=useState()

    useEffect(()=>{
        if(session_id) setSessionId(session_id);
        else{
            if(localStorage.session_id) setSessionId(localStorage.session_id)
        }
    },[])

    useEffect(()=>{
        if(!localStorage.session_id) localStorage.setItem('session_id',sessionId)
    },[sessionId])

    return{
        sessionId
    }
}
```

So now whenever you need to use session ID on a particular page, you can grab it from this hook. Thus, you won't need to pass it as query parameters in your React app's routes. 


#### 3. CSRF
* CSRF is definitely a concern that we want to address with our app. A good solution is the use of CSRF Tokens. A good library to use, assuming you have a NodeJS and Express back end that interacts with your React client, is [csuf](https://www.npmjs.com/package/csurf). 

We can create an GET endpoint to fetch the generated CSRFToken from our Express backend. 

```javascript
const csrfProtection = csrf({
  cookie: true
});
app.use(csrfProtection);
app.get('/getCSRFToken', (req, res) => {
  res.json({ CSRFToken: req.CSRFToken() });
});
```

Then have our client app fetch the token and attach it to every request! 
```javascript
const getCSRFToken = async () => {
    const response = await axios.get('/getCSRFToken');
    axios.defaults.headers.post['X-CSRF-Token'] = response.data.CSRFToken;
 };
```


