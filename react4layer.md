# React 4-Layer Architecture — API → Service → Hooks → UI

Ek complete guide jisme dekhenge ki React app ko 4 clean layers mein kaise organize karte hain, taaki code scalable, testable aur maintainable rahe.

---

## 1. Pura Flow Ek Nazar Mein

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  API SERVICE │ ---> │    REDUX     │ ---> │    HOOKS     │ ---> │      UI      │
│              │      │   (slice)    │      │              │      │              │
│ raw fetch/   │      │ state ka     │      │ dispatch +   │      │ user ko      │
│ axios calls  │      │ blueprint +  │      │ api call +   │      │ dikhne wala  │
│ backend ko   │      │ reducers     │      │ side effects │      │ form/button  │
└──────────────┘      └──────────────┘      └──────────────┘      └──────────────┘
      Layer 1               Layer 2               Layer 3               Layer 4
   "Data kaise         "Data kahan          "Kab aur kaise         "Data kaise
    mangna hai"          store hoga"          trigger karna"         dikhana hai"
```

**Golden rule:** UI kabhi seedha fetch nahi karta, aur service kabhi Redux ko nahi jaanta. Beech mein hamesha hook hota hai jo dono ko connect karta hai.

---

## 2. Layer 1 — API Service Layer

Iska sirf ek kaam hai: **backend se baat karna**. Is layer ko koi Redux nahi pata, koi component nahi pata — bas plain functions hote hain jo `fetch` ya `axios` se request bhejte hain aur response return karte hain.

```js
// auth/service/auth.api.js

export async function login(email, password) {
  const res = await fetch('/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
  });
  if (!res.ok) throw new Error('Login failed');
  return res.json(); // { user, token }
}

export async function getme() {
  const res = await fetch('/api/auth/me', { credentials: 'include' });
  if (!res.ok) throw new Error('Not authenticated');
  return res.json();
}
```

**Rule:** is layer mein koi `dispatch` nahi hota, koi `useState` nahi hota, koi React import nahi hota. Pure JS functions. Agar kal backend ka URL badal jaaye ya `axios` pe switch karna ho, sirf isi file mein change hoga — baaki poora app untouched rahega.

---

## 3. Layer 2 — Redux Slice (State ka Blueprint)

Yeh layer decide karti hai ki app ke andar **data kis shape mein store hoga**, aur usse update karne ke legal tareeke (actions) kya hain. Slice khud kabhi API call nahi karti — woh bas result receive karke state update karti hai.

```js
// auth/auth.slice.js

const initialState = {
  user: null,
  token: null,
  isAuthenticated: false,
  loading: true,
  error: null,
};

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    authStart: (state) => {
      state.loading = true;
      state.error = null;
    },
    authSuccess: (state, action) => {
      const { user, token = null } = action.payload || {};
      state.loading = false;
      state.isAuthenticated = Boolean(user);
      state.user = user ?? state.user;
      state.token = token ?? state.token ?? null;
    },
    authFailure: (state, action) => {
      state.loading = false;
      state.isAuthenticated = false;
      state.user = null;
      state.token = null;
      state.error = action.payload;
    },
    authLogout: (state) => { /* reset everything */ },
  },
});

export const { authStart, authSuccess, authFailure, authLogout } = authSlice.actions;
export default authSlice.reducer;
```

Sab slices ek jagah `store.js` mein combine hote hain:

```js
const store = configureStore({
  reducer: {
    auth: authReducer,
    products: productReducer,
    cart: cartReducer,
    search: searchReducer,
  },
});
```

Socho slice ek **form/register** jaisa hai jisme columns already defined hain (`user`, `token`, `loading`, `error`). Jab bhi koi action fire hoti hai, Redux us form ke fields update kar deta hai — aur jo component us state ko sun raha hai (`useSelector` se), woh automatically re-render ho jaata hai.

---

## 4. Layer 3 — Custom Hook (Service aur Redux ke Beech ka Pul)

Yeh layer Layer 1 (API) aur Layer 2 (Redux) ko **jodti** hai. Component ko yeh nahi pata hona chahiye ki andar fetch ho raha hai ya dispatch — component ko bas ek clean function chahiye jaise `handlelogin(email, password)`.

```js
// auth/hook/UseAuth.js

const useauth = () => {
  const dispatch = useDispatch();

  async function handlelogin(email, password) {
    dispatch(authStart());                         // 1) Redux ko bolo "loading shuru"
    try {
      const data = await login(email, password);   // 2) API service ko call karo
      dispatch(authSuccess({                        // 3) success pe Redux update karo
        user: data.user ?? null,
        token: data.token ?? null
      }));
      return { success: true, data };               // 4) component ko result do
    } catch (error) {
      const message = error?.message || 'Login failed';
      dispatch(authFailure(message));               // 3b) fail pe error store karo
      return { success: false, error: message };
    }
  }

  return { handlelogin /*, handleregister, handlegetme, handleGoogleVerify */ };
};

export default useauth;
```

Yahan ek **same pattern** repeat hota hai har function mein:

```
dispatch(start) → try API call → dispatch(success) → catch → dispatch(failure)
```

Yeh pattern hi is layer ki pehchaan hai.

---

## 5. Layer 4 — UI Component

Sabse upar wali layer — jo user dekhta hai. Iska kaam sirf **hook ko call karna** aur **state ke hisaab se UI dikhana** hai. Yahan koi fetch, koi dispatch directly nahi hota.

```jsx
// Login.jsx

const { handlelogin } = useauth();   // sirf hook ko call kiya

const handleSubmit = async (e) => {
  e.preventDefault();
  const res = await handlelogin(email, password);   // clean function call

  if (res.success) {
    navigate('/dashboard');
  }
};
```

Aur kisi bhi component mein Redux state seedha `useSelector` se padh sakte hain:

```jsx
const loading = useSelector((state) => state.auth?.loading);
const user = useSelector((state) => state.auth?.user);

if (loading && !user) {
  return <div>Loading account...</div>;
}
```

Component ko yeh pata hi nahi ki `login()` ke andar fetch ho raha hai ya kaunsa endpoint hit ho raha hai. Component ka kaam bas itna: hook call karo, result ke hisaab se navigate ya render karo.

---

## 6. Ek Real Click Ka Poora Safar

Jab user 'Sign In' button dabata hai, data 4 layers se hote hue kaise travel karta hai:

| # | Layer | Kya hota hai |
|---|-------|---------------|
| 1 | UI | `handleSubmit` fire hota hai → `handlelogin(email, password)` call hota hai |
| 2 | Hook | `dispatch(authStart())` — Redux ko batata hai "loading shuru" |
| 3 | Hook → Service | `await login(email, password)` call hota hai (auth.api.js ka function) |
| 4 | Service | `fetch()` backend ko POST request bhejta hai, response JSON return karta hai |
| 5 | Hook | response milte hi `dispatch(authSuccess({ user, token }))` |
| 6 | Redux Slice | `authSuccess` reducer chalta hai — `state.user`, `state.isAuthenticated` set hote hain |
| 7 | UI | `useSelector` wale components automatically re-render, `navigate('/dashboard')` chalta hai |

---

## 7. Yeh 4 Layers Banane Ki Zaroorat Kyun?

- **Testing easy ho jaata hai** — API service ko akela test kar sakte ho bina kisi component ya Redux ke.
- **Backend badalna aasan** — REST se GraphQL pe jaana ho, sirf Layer 1 badlega, Layer 2/3/4 untouched.
- **Reusability** — `handlelogin()` ko kisi bhi component mein reuse kar sakte ho.
- **Component saaf rehta hai** — UI mein koi fetch/dispatch logic nahi, sirf UI aur interaction.
- **Debug karna aasan** — data galat hai toh Layer 1/2 dekho, UI galat render ho raha hai toh Layer 4 dekho.

---

## 8. Folder Structure

```
src/
├── store.js                     <- Layer 2: sab slices ek jagah combine
├── auth/
│   ├── auth.slice.js            <- Layer 2: auth ka state + reducers
│   ├── service/
│   │   └── auth.api.js          <- Layer 1: raw fetch calls
│   ├── hook/
│   │   └── UseAuth.js           <- Layer 3: service + redux ka pul
│   └── Login.jsx                <- Layer 4: UI
├── cart/
│   └── cart.slice.js
└── components/
    └── search.slice.js
```

Jo pattern auth ke liye use hua hai, wahi pattern cart, products, search ke liye bhi follow hota hai — har feature ka apna service + slice + hook + UI. Isse pura app predictable rehta hai, chahe kitna bhi bada ho jaaye.

---

## 9. Quick Recap Table

| Layer | Sawaal jo yeh answer karti hai | Import karti hai kya? |
|-------|----------------------------------|--------------------------|
| 1. API Service | Backend se data kaise mangu? | sirf fetch/axios |
| 2. Redux Slice | Data kis shape mein rahega? | `@reduxjs/toolkit` |
| 3. Hook | Kab aur kis order mein trigger karu? | Layer 1 + Layer 2 (dispatch) |
| 4. UI | User ko kya dikhau? | sirf Layer 3 ka hook |

**Golden rule (dobara):** UI kabhi seedha fetch nahi karta, aur service kabhi Redux ko nahi jaanta. Beech mein hamesha hook hota hai jo dono ko connect karta hai.
