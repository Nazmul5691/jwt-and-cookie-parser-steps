
# 🔐 JWT Authentication with Cookies (Server + Client Setup)

This guide walks through the complete implementation of JWT-based authentication using `jsonwebtoken` and `cookie-parser`, from backend token creation to frontend protected route access.

---

## 🔧 1. Install Dependencies

```bash
npm install jsonwebtoken cookie-parser
```

---

## 📥 2. Import and Use Middleware

In your `server.js` or `index.js`:

```js
const jwt = require('jsonwebtoken');
const cookieParser = require('cookie-parser');

app.use(cookieParser());
```

---

## 🔑 3. Create a JWT Token Endpoint

```js
app.post('/jwt', async (req, res) => {
  const user = req.body;
  const token = jwt.sign(user, process.env.ACCESS_TOKEN_SECRET, { expiresIn: '10h' });

  res
    .cookie('token', token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production", // false for localhost
      sameSite: process.env.NODE_ENV === "production" ? "none" : "strict", //There is no error if you don't use it in localhost. But must needed for production.
    })
    .send({ success: true });
});
```

---

## 🌍 4. Set CORS Configuration

```js
app.use(cors({
  origin: ['http://localhost:5173'],
  credentials: true
}));
```

---

## 🧠 5. Create a Custom Axios Hook (Client Side)

File: `hooks/UseAxiosSecure.js`

```js
import axios from "axios";
import { useEffect } from "react";
import UseAuth from "./UseAuth";
import { useNavigate } from "react-router-dom";

const axiosInstance = axios.create({
  baseURL: 'https://job-portal-server-lilac-phi.vercel.app',
  withCredentials: true
});

const UseAxiosSecure = () => {
  const { logOutUser } = UseAuth();
  const navigate = useNavigate();

  useEffect(() => {
    axiosInstance.interceptors.response.use(response => {
      return response;
    }, error => {
      if (error.status === 401 || error.status === 403) {
        logOutUser()
          .then(() => navigate('/signIn'))
          .catch(err => console.log(err));
      }
      return Promise.reject(error);
    });
  }, []);

  return axiosInstance;
};

export default UseAxiosSecure;
```

---

## 📦 6. Use Hook in the selected Component for use token and verifyToken (Client Side)

File: `MyApplications.jsx`

```js
import { useEffect, useState } from "react";
import UseAuth from "../hooks/UseAuth";
import UseAxiosSecure from "../hooks/UseAxiosSecure";

const MyApplications = () => {
  const { user } = UseAuth();
  const [jobs, setJobs] = useState([]);
  const axiosSecure = UseAxiosSecure([]);

  useEffect(() => {
    //for use token and verifyToken
    axiosSecure.get(`/job-applications?email=${user.email}`)
      .then(res => setJobs(res.data));
  }, [user.email]);

  return (
    <div>
      <h1>My applications: {jobs.length}</h1>
    </div>
  );
};

export default MyApplications;
```

---

## 👤 7. Generate JWT on Auth Change

File: `AuthProvider.jsx`

```js
useEffect(()=>{
        const unsubscribe = onAuthStateChanged(auth, currentUser =>{
            setUser(currentUser)
            console.log(currentUser);
            //for create token
            if(currentUser?.email){
                const user = { email : currentUser.email}
                axios.post('https://job-portal-server-lilac-phi.vercel.app/jwt', user, {withCredentials: true})
                .then(res => {
                    console.log(res.data)
                    setLoading(false)
                })
            }
            else{
                axios.post('https://job-portal-server-lilac-phi.vercel.app/logout',{} , {withCredentials: true})
                .then(res => {
                    console.log(res.data)
                    setLoading(false)
                })
            } 
        })
        return() =>{
            unsubscribe()
        }
    },[])
```

Full code of AuthProvider.jsx

```js
import { useEffect, useState } from "react";
import AuthContext from "./AuthContext";
import { createUserWithEmailAndPassword, GoogleAuthProvider, onAuthStateChanged, signInWithEmailAndPassword, signInWithPopup, signOut } from "firebase/auth";
import { auth } from "../firebase/firebase.init";
import axios from "axios";

const googleProvider = new GoogleAuthProvider();

const AuthProvider = ({children}) => {

    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    const createUser = (email, password) => {
        setLoading(true)
        return createUserWithEmailAndPassword(auth, email, password)
    }

    const signInUser = (email, password) =>{
        setLoading(true)
        return signInWithEmailAndPassword(auth, email, password)
    }

    const socialLogIn = () =>{
        setLoading(true)
        return signInWithPopup(auth, googleProvider)
    }

    const logOutUser = () =>{
        setLoading(true)
        return signOut(auth)
    }

    useEffect(()=>{
        const unsubscribe = onAuthStateChanged(auth, currentUser =>{
            setUser(currentUser)
            console.log(currentUser);
            //for create token
            if(currentUser?.email){
                const user = { email : currentUser.email}
                axios.post('https://job-portal-server-lilac-phi.vercel.app/jwt', user, {withCredentials: true})
                .then(res => {
                    console.log(res.data)
                    setLoading(false)
                })
            }
            else{
                axios.post('https://job-portal-server-lilac-phi.vercel.app/logout',{} , {withCredentials: true})
                .then(res => {
                    console.log(res.data)
                    setLoading(false)
                })
            }
            
            
        })
        return() =>{
            unsubscribe()
        }
    },[])


    const userInfo = {
        user,
        loading,
        createUser,
        signInUser,
        socialLogIn,
        logOutUser
    }

    return (
        <AuthContext.Provider value={userInfo}>
            {children}
        </AuthContext.Provider>
    );
};

export default AuthProvider;
```

---

## 🛂 8. Update SignIn Logic

File: `SignIn.jsx`

```js
signInUser(email, password)
  .then(result => {
    const user = { email };
    axios.post('https://job-portal-server-lilac-phi.vercel.app/jwt', user, { withCredentials: true })
      .then(() => navigate(from));
  })
  .catch(error => console.log(error.message));
```

full signIn code

```js
import { Form, useLocation, useNavigate } from "react-router-dom";
import registration from '../assets/registration.json'
import Lottie from "lottie-react";
import { useContext } from "react";
import AuthContext from "../context/AuthContext";
import SocialLogin from "../shared/SocialLogin";
import axios from "axios";


const SignIn = () => {

    const { signInUser } = useContext(AuthContext)
    
    const navigate = useNavigate();
    const location = useLocation()
    const from = location?.state || '/'



    const handleSignIn = e => {
        e.preventDefault()

        const form = e.target
        const email = form.email.value;
        const password = form.password.value;

        console.log(email, password);

        signInUser(email, password)
            .then(result => {
                console.log('sign in user', result.user);

                const user = {email: email}

                axios.post('https://job-portal-server-lilac-phi.vercel.app/jwt',user, {withCredentials: true})
                .then(res =>{
                    console.log(res.data);
                })
                navigate(from)
            })
            .catch(error => {
                console.log(error.message);
            })
    }


    return (
        <div>

        </div>
    );
};

export default SignIn;
```
---

## 🔐 9. Verify Token on Backend

```js
const verifyToken = (req, res, next) => {
  const token = req?.cookies?.token;

  if (!token) {
    return res.status(401).send({ message: 'unauthorized access' });
  }

  jwt.verify(token, process.env.ACCESS_TOKEN_SECRET, (err, decoded) => {
    if (err) {
      return res.status(401).send({ message: 'unauthorized access' });
    }
    req.user = decoded;
    next();
  });
};
```

---

## 🔎 10. Use `verifyToken` in Protected Route

```js
app.get('/job-applications', verifyToken, async (req, res) => {
  const email = req.query.email;
  const query = { applicant_email: email };

      // for cookie
      // console.log('cuk cuk cookie', req.cookies);
      if (req.user?.email !== req.query?.email) {
        return res.status(403).send({ message: 'forbidden access' })
      }

  const result = await jobsApplicationCollection.find(query).toArray();

  for (const application of result) {
    const job = await jobsCollection.findOne({ _id: new ObjectId(application.job_id) });
    if (job) {
      application.title = job.title;
      application.company = job.company;
      application.company_logo = job.company_logo;
      application.location = job.location;
    }
  }

  res.send(result);
});
```

---

✅ You're now fully set up with JWT, secure cookies, and protected client-side data fetching!
















<!-- jwt and cookie parser steps

1. install jsonwebtoken cookie-parser
npm i jsonwebtoken cookie-parser

2. set require and const 
const jwt = require('jsonwebtoken')
const cookieParser = require('cookie-parser')

3. set cookie-parser as middleware
app.use(cookieParser())

4. create a token 
app.post('/jwt', async (req, res) => {
      const user = req.body;
      const token = jwt.sign(user, process.env.ACCESS_TOKEN_SECRET, { expiresIn: '10h' })


    //set token to the cookies in application
      res
        .cookie('token', token, {
          httpOnly: true,
          secure: process.env.NODE_ENV === "production", //use secure: false for localhost
          sameSite: process.env.NODE_ENV === "production" ? "none" : "strict", // this is use for production
        })
        .send({ success: true })
    })


5. make origin in cors 
app.use(cors({
  origin: ['http://localhost:5173'],
  credentials: true
}))

6. make a custom hook in client side for load URL like -

import axios from "axios";
import { useEffect } from "react";
import UseAuth from "./UseAuth";
import { useNavigate } from "react-router-dom";

const axiosInstance = axios.create({
    baseURL: 'https://job-portal-server-lilac-phi.vercel.app',
    withCredentials: true
})

const UseAxiosSecure = () => {


    const {logOutUser} = UseAuth();
    const navigate = useNavigate()

    useEffect(()=>{
        axiosInstance.interceptors.response.use(response =>{
            return response;
        }, error =>{
            if(error.status === 401 || error.status === 403){
                logOutUser()
                .then(res =>{
                    navigate('/signIn')
                })
                .catch(err => console.log(err))
            }
            return Promise.reject(error)
        })
    },[])


    return axiosInstance;
};

export default UseAxiosSecure;

see this in job-portal-client-site at github/Nazmul5691

7. now use this in client side selected route , where you want to use this

useEffect(() => {
        // axios.get(`https://job-portal-server-lilac-phi.vercel.app/job-applications?email=${user.email}`,{withCredentials: true},
        // )
            axiosSecure.get(`/job-applications?email=${user.email}`)
            .then(res => setJobs(res.data))
    }, [user.email])



full code of file

import { useEffect, useState } from "react";
import UseAuth from "../hooks/UseAuth";
import { Link } from "react-router-dom";
import axios from "axios";
import UseAxiosSecure from "../hooks/UseAxiosSecure";


const MyApplications = () => {
    const { user } = UseAuth()
    const [jobs, setJobs] = useState([])

    const axiosSecure = UseAxiosSecure([])

    useEffect(() => {
            axiosSecure.get(`/job-applications?email=${user.email}`)
            .then(res => setJobs(res.data))
    }, [user.email])

    return (
        <div>
            <h1>my applications : {jobs.length}</h1>
        </div>
    );
};

export default MyApplications;

8. create a token in onAuthStateChange in authProvider.jsx

import { useEffect, useState } from "react";
import AuthContext from "./AuthContext";
import { createUserWithEmailAndPassword, GoogleAuthProvider, onAuthStateChanged, signInWithEmailAndPassword, signInWithPopup, signOut } from "firebase/auth";
import { auth } from "../firebase/firebase.init";
import axios from "axios";

const googleProvider = new GoogleAuthProvider();

const AuthProvider = ({children}) => {

    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    const createUser = (email, password) => {
        setLoading(true)
        return createUserWithEmailAndPassword(auth, email, password)
    }

    const signInUser = (email, password) =>{
        setLoading(true)
        return signInWithEmailAndPassword(auth, email, password)
    }

    const socialLogIn = () =>{
        setLoading(true)
        return signInWithPopup(auth, googleProvider)
    }

    const logOutUser = () =>{
        setLoading(true)
        return signOut(auth)
    }

    useEffect(()=>{
        const unsubscribe = onAuthStateChanged(auth, currentUser =>{
            setUser(currentUser)
            console.log(currentUser);
            //for create token
            if(currentUser?.email){
                const user = { email : currentUser.email}
                axios.post('https://job-portal-server-lilac-phi.vercel.app/jwt', user, {withCredentials: true})
                .then(res => {
                    console.log(res.data)
                    setLoading(false)
                })
            }
            else{
                axios.post('https://job-portal-server-lilac-phi.vercel.app/logout',{} , {withCredentials: true})
                .then(res => {
                    console.log(res.data)
                    setLoading(false)
                })
            }
            
            
        })
        return() =>{
            unsubscribe()
        }
    },[])

    const userInfo = {
        user,
        loading,
        createUser,
        signInUser,
        socialLogIn,
        logOutUser
    }

    return (
        <AuthContext.Provider value={userInfo}>
            {children}
        </AuthContext.Provider>
    );
};

export default AuthProvider;

9. update sign in code 
signInUser(email, password)
            .then(result => {
                console.log('sign in user', result.user);

                const user = {email: email}

                axios.post('https://job-portal-server-lilac-phi.vercel.app/jwt',user, {withCredentials: true})
                .then(res =>{
                    console.log(res.data);
                })
                navigate(from)
            })
            .catch(error => {
                console.log(error.message);
            })

full code of signIn.jsx

import { Form, useLocation, useNavigate } from "react-router-dom";
import registration from '../assets/registration.json'
import Lottie from "lottie-react";
import { useContext } from "react";
import AuthContext from "../context/AuthContext";
import SocialLogin from "../shared/SocialLogin";
import axios from "axios";


const SignIn = () => {

    const { signInUser } = useContext(AuthContext)
    
    const navigate = useNavigate();
    const location = useLocation()
    const from = location?.state || '/'



    const handleSignIn = e => {
        e.preventDefault()

        const form = e.target
        const email = form.email.value;
        const password = form.password.value;

        console.log(email, password);

        signInUser(email, password)
            .then(result => {
                console.log('sign in user', result.user);

                const user = {email: email}

                axios.post('https://job-portal-server-lilac-phi.vercel.app/jwt',user, {withCredentials: true})
                .then(res =>{
                    console.log(res.data);
                })
                navigate(from)
            })
            .catch(error => {
                console.log(error.message);
            })
    }


    return (
        <div>
        </div>
    );
};

export default SignIn;


10. verify the token in sever side 

const verifyToken = (req, res, next) => {
  const token = req?.cookies?.token
  // console.log('inside the verify token', token);

  if (!token) {
    return res.status(401).send({ message: 'unauthorized access' })
  }

  // verify token
  jwt.verify(token, process.env.ACCESS_TOKEN_SECRET, (err, decoded) => {
    if (err) {
      return res.status(401).send({ message: 'unauthorized access' })
    }
    req.user = decoded
    next()
  })
}

11. use verifyToken in api

app.get('/job-applications', verifyToken, async (req, res) => {
      const email = req.query.email
      const query = { applicant_email: email }


      // for check cookie
      // console.log('cuk cuk cookie', req.cookies);

      if (req.user?.email !== req.query?.email) {
        return res.status(403).send({ message: 'forbidden access' })
      }


      const result = await jobsApplicationCollection.find(query).toArray()

      for (const application of result) {
        // console.log(application.job_id);
        const query1 = { _id: new ObjectId(application.job_id) }
        const result1 = await jobsCollection.findOne(query1)
        if (result1) {
          application.title = result1.title
          application.company = result1.company
          application.company_logo = result1.company_logo
          application.location = result1.location
        }
      }

      res.send(result)
    })





 -->
