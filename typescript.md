# TypeScript for JS Developers — Zero to Hero
### (Tumhe JS aata hai, ab sirf "naya" wala TS seekhna hai)

> Rule jo hum follow karenge: jo bhi JS mein same hai (variable declare karna, loops, functions ka basic syntax) — usko hum sirf ek line mein touch karenge. Jo bhi TS-only concept hai (types, union, enum, generics, interfaces) — uska full definition + example + JS-developer-analogy milega.

---

## Table of Contents
1. TypeScript kya hai aur kyu chahiye
2. Setup (compiler, config, run karna)
3. Variables — kya same hai, kya naya hai
4. Basic Types (primitives ke saath type annotation)
5. Arrays & Tuples
6. Type Inference
7. `any`, `unknown`, `never`, `void`
8. **Union Types**
9. **Intersection Types**
10. Literal Types
11. **Enums**
12. Functions in TS
13. Objects & Interfaces
14. `type` vs `interface`
15. Classes in TS
16. Generics
17. Type Narrowing
18. Utility Types
19. Type Assertion
20. Modules (import/export ka TS version)
21. TypeScript + React (practical — Sketch AI jaisa project)
22. TypeScript + Node/Express (practical — backend wala)
23. `tsconfig.json` deep dive
24. Best Practices + Common Mistakes
25. Mini Practice Project + Exercises

---

## 1. TypeScript kya hai aur kyu chahiye

JavaScript **dynamically typed** language hai — matlab variable ka type runtime pe decide hota hai, aur galti tabhi pata chalti hai jab code chal jaaye.

```js
function add(a, b) {
  return a + b;
}
add(5, "10"); // "510" — bug, par JS chup hai
```

TypeScript = **JavaScript + Type System**. Yeh ek **superset** hai — jo bhi `.js` mein chalta hai, wahi `.ts` mein bhi valid hai. TS sirf compile-time pe ek extra layer add karta hai jo type-mismatch ko **code likhte waqt hi** pakad leta hai (editor mein laal underline dikhega).

```ts
function add(a: number, b: number): number {
  return a + b;
}
add(5, "10"); // Compile-time error: Argument of type 'string' is not assignable to parameter of type 'number'
```

**Important:** Browser TypeScript ko directly nahi samajhta. TS code `tsc` (TypeScript Compiler) se compile hokar plain JS mein convert hota hai, aur woh JS hi run hota hai. Toh TS sirf development-time ka safety net hai, runtime pe kuch extra nahi karta.

---

## 2. Setup

```bash
npm install -g typescript        # globally compiler install
tsc --version                    # check

# project mein local install (recommended)
npm install --save-dev typescript

# config file generate karo
npx tsc --init
```

Iske baad ek `tsconfig.json` ban jayega jo bata deta hai TS compiler ko ki kaise compile karna hai (detail Chapter 23 mein).

Quick run karne ke liye `ts-node` use karo (bina manually compile kiye directly `.ts` run karne ke liye):

```bash
npm install -g ts-node
ts-node file.ts
ts-node app.ts ; npx tsc app.ts --ignoreConfig // to create app.js file and run ts file
```

---

## 3. Variables — kya same hai, kya naya hai

`let`, `const`, `var` — **bilkul same** hai jaisa JS mein hai. Scoping rules, hoisting, sab kuch same. Yeh part repeat nahi karunga.

**Jo naya hai:** variable declare karte waqt uska **type bhi specify** kar sakte ho (aur recommended bhi hai). Yeh syntax hai:

```ts
let variableName: Type = value;
```

```ts
let username: string = "Anand";
let age: number = 21;
let isLoggedIn: boolean = true;
```

Bas yahi naya hai — colon (`:`) ke baad type likhna. Value assign karna, `=` ka use, sab same hai jaisa JS mein hota hai.

---

## 4. Basic Types

| Type | Example |
|---|---|
| `string` | `let name: string = "Anand"` |
| `number` | `let age: number = 21` (int/float dono `number` hi hain, JS jaisa) |
| `boolean` | `let active: boolean = true` |
| `null` | `let x: null = null` |
| `undefined` | `let y: undefined = undefined` |
| `bigint` | `let big: bigint = 100n` |
| `symbol` | `let sym: symbol = Symbol("id")` |

```ts
let city: string = "Ranchi";
let pin: number = 834001;
let isOnline: boolean = false;
```

> Agar value already assign kar rahe ho, TS automatically type samajh leta hai (Chapter 6 dekho — Type Inference). Toh `let pin: number = 834001` likhna zaroori nahi, `let pin = 834001` se bhi kaam ho jayega. Type likhna optional hai par **function params aur return types mein hamesha likhna chahiye** — best practice.

---

## 5. Arrays & Tuples

### Array

JS mein array ka type fix nahi hota — same array mein number, string, object sab daal sakte ho. TS mein array ka ek fixed element-type hota hai.

```ts
let scores: number[] = [10, 20, 30];
let names: string[] = ["Anand", "Rahul"];

// generic syntax (same matlab)
let scores2: Array<number> = [10, 20, 30];
```

Agar mixed type chahiye:

```ts
let mixed: (string | number)[] = ["Anand", 21, "Ranchi"];
```

### Tuple — yeh JS mein nahi hota

Tuple ek **fixed-length, fixed-order** array hai jahan har position ka type predetermined hota hai.

```ts
let user: [string, number] = ["Anand", 21]; // [name, age] — order matter karta hai
user[0] = "Rahul";  // ✅ allowed (string)
user[1] = "21";     // ❌ error, number expected
```

Real use-case: jaise `useState` return karta hai `[value, setValue]` — yeh internally tuple hi hai.

---

## 6. Type Inference

TS itna smart hai ki value se khud type guess kar leta hai — explicit type likhna zaroori nahi har jagah.

```ts
let city = "Ranchi";   // TS samajh gaya: type string hai
city = 123;             // ❌ error — even bina explicit type likhe!
```

Inference best hai jab value initialize ke saath declare ho rahi ho. Jab variable bina value declare karo (jaise function parameter), tab explicit type **zaroori** ho jaata hai kyunki TS guess nahi kar sakta.

---

## 7. `any`, `unknown`, `never`, `void`

JS developer ke liye yeh 4 sabse confusing lagte hain, isliye clear definition + example:

### `any` — type-checking off kar deta hai
```ts
let data: any = 5;
data = "hello";   // ✅ allowed
data = true;      // ✅ allowed, koi error nahi
```
`any` matlab TS ne us variable ke liye **type-safety chhod di**. Yeh escape-hatch hai — jitna kam use karo, behtar. Agar har jagah `any` use kar rahe ho, toh TypeScript likhne ka fayda hi nahi hai.

### `unknown` — `any` ka safe version
```ts
let data: unknown = 5;
data = "hello";          // ✅ assign karna allowed
console.log(data.length); // ❌ error — use karne se pehle type check karna padega
```
```ts
if (typeof data === "string") {
  console.log(data.length); // ✅ ab allowed, TS ko pata hai yeh string hai
}
```
Jab tumhe pata nahi value kya aayegi (jaise API response), `unknown` use karo, `any` nahi.

### `never` — wo type jo kabhi value le hi nahi sakta
```ts
function throwError(msg: string): never {
  throw new Error(msg);
}
```
Function jo kabhi return hi nahi hota (ya infinite loop, ya hamesha error throw karta hai) uska return type `never` hota hai. Real use: discriminated union ke exhaustive check mein (Chapter 17 mein dekhoge).

### `void` — function kuch return nahi karta
```ts
function logMessage(msg: string): void {
  console.log(msg);
  // kuch return nahi ho raha
}
```
JS mein bhi function `undefined` return karta hai agar `return` na likho. `void` batata hai "iska return value use mat karo, intentional hai".

---

## 8. Union Types — JS mein yeh exist nahi karta

**Definition:** Union type ka matlab hai — ek variable/parameter **multiple types mein se koi ek** ho sakta hai. Pipe (`|`) symbol use hota hai.

```ts
let id: string | number;

id = 101;       // ✅
id = "ABC123";  // ✅
id = true;      // ❌ error — boolean allowed nahi
```

### Function parameter mein union

```ts
function printId(id: string | number) {
  console.log("Your ID is: " + id);
}

printId(101);     // ✅
printId("U-101"); // ✅
```

### Union ke saath kaam karne ke liye narrowing zaroori hai

```ts
function formatId(id: string | number) {
  if (typeof id === "string") {
    return id.toUpperCase();   // yahan TS ko pata hai id string hai
  }
  return id.toFixed(2);        // yahan TS ko pata hai id number hai
}
```
(Narrowing ka full chapter aage hai — Chapter 17)

### Union of object types — bohot common pattern

```ts
type Cat = { type: "cat"; meow: () => void };
type Dog = { type: "dog"; bark: () => void };

type Pet = Cat | Dog;

function makeSound(pet: Pet) {
  if (pet.type === "cat") {
    pet.meow();
  } else {
    pet.bark();
  }
}
```

**Analogy:** Socho ek function ko ek door pe khada karo jo sirf "Cat ya Dog" entry allow karta hai, "Elephant" allow nahi. Union type wahi gatekeeper hai.

---

## 9. Intersection Types — yeh bhi JS mein nahi hota

**Definition:** Intersection type ka matlab — ek type **multiple types ke saare properties combine karke** banta hai. `&` symbol use hota hai. Union "ya toh A ya toh B" hai, Intersection "A AND B dono" hai.

```ts
type Person = {
  name: string;
};

type Employee = {
  employeeId: number;
};

type Staff = Person & Employee; // dono ke properties chahiye

const staff1: Staff = {
  name: "Anand",
  employeeId: 101,
}; // dono fields zaroori hain, ek bhi missing = error
```

**Real use-case:** React mein jab ek component ke props mein default HTML attributes + custom props dono chahiye hote hain:

```ts
type ButtonProps = {
  label: string;
} & React.ButtonHTMLAttributes<HTMLButtonElement>;
```

| | Union (`|`) | Intersection (`&`) |
|---|---|---|
| Matlab | Ek ya doosra | Dono ek saath |
| Example | `string \| number` | `Person & Employee` |
| Object case | "Cat ya Dog, koi ek" | "Person ke + Employee ke saare fields chahiye" |

---

## 10. Literal Types

Sirf specific fixed values allow karna — type ke andar exact value daal dete ho.

```ts
let direction: "left" | "right" | "up" | "down";

direction = "left";   // ✅
direction = "forward"; // ❌ error, sirf 4 values allowed
```

Yeh union type ka hi extension hai, par values ke "set" ko restrict karta hai. Real use-case: API status, button variant, theme mode jaisi fixed-choice cheezein.

```ts
type Theme = "light" | "dark";
function setTheme(theme: Theme) { /* ... */ }
```

---

## 11. Enums — JS mein bilkul nahi hota

**Definition:** Enum (Enumeration) ek tarah ka named-constant set hai — jab tumhare paas fixed, related values ka group ho (jaise days, status, roles), enum unko **readable names** deta hai.

JS mein log aise karte hain:
```js
const STATUS_PENDING = 0;
const STATUS_SUCCESS = 1;
const STATUS_FAILED = 2;
```
TS mein enum se yeh cleaner ho jaata hai:

### Numeric Enum
```ts
enum Status {
  Pending,   // 0 (default, auto increment)
  Success,   // 1
  Failed,    // 2
}

let orderStatus: Status = Status.Success;
console.log(orderStatus); // 1
```
Tum starting number bhi specify kar sakte ho:
```ts
enum Status {
  Pending = 1,
  Success,  // 2 (auto continue)
  Failed,   // 3
}
```

### String Enum — production mein zyada use hota hai (debugging easy)
```ts
enum Role {
  Admin = "ADMIN",
  User = "USER",
  Guest = "GUEST",
}

function checkAccess(role: Role) {
  if (role === Role.Admin) {
    console.log("Full access");
  }
}

checkAccess(Role.Admin); // "ADMIN"
```
String enum better hai numeric se kyunki `console.log` karne pe ya debugger mein dekhne pe `"ADMIN"` dikhega, `0`/`1` nahi — readable rehta hai.

### `const enum` — performance-friendly version
```ts
const enum Direction {
  Up,
  Down,
}
let d = Direction.Up;
```
`const enum` compile hone ke baad code mein gayab ho jaata hai — directly value inline ho jaati hai (`0`), JS object generate nahi hota. Bundle size thoda chhota rehta hai.

### Enum vs Union Literal — kab kya use karein?

```ts
// Option A: Enum
enum Theme { Light = "LIGHT", Dark = "DARK" }

// Option B: Union of literals
type Theme = "LIGHT" | "DARK";
```
Modern TS codebases (especially React projects) mein **union literal types zyada prefer** kiye jaate hain kyunki yeh extra JS code generate nahi karte aur tree-shaking friendly hain. Enum tab use karo jab tumhe values pe methods/iteration chahiye ho, warna union literal hi zyada lightweight option hai.

---

## 12. Functions in TS

Parameters aur return type dono ko type kar sakte ho:

```ts
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

### Optional Parameter (`?`)
```ts
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}`;
}
greet("Anand");              // ✅
greet("Anand", "Namaste");   // ✅
```

### Default Parameter
```ts
function greet(name: string, greeting: string = "Hello") {
  return `${greeting}, ${name}`;
}
```

### Rest Parameter
```ts
function sum(...nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0);
}
```

### Function Type (variable ko function ka "shape" dena)
```ts
let add: (a: number, b: number) => number;
add = (a, b) => a + b;
```

---

## 13. Objects & Interfaces

JS mein object literal banate ho without any shape-restriction. TS mein object ka shape define kar sakte ho:

```ts
let user: { name: string; age: number } = {
  name: "Anand",
  age: 21,
};
```

Yeh repeat karna padega har jagah, isliye **interface** banate hain — ek reusable shape/blueprint:

```ts
interface User {
  name: string;
  age: number;
  email?: string;       // optional property
  readonly id: number;  // ek baar set hone ke baad change nahi ho sakta
}

const u1: User = { id: 1, name: "Anand", age: 21 };
u1.id = 2; // ❌ error — readonly hai
```

### Interface mein functions
```ts
interface User {
  name: string;
  greet(): string;
}

const u2: User = {
  name: "Anand",
  greet() {
    return `Hi, I'm ${this.name}`;
  },
};
```

### Interface Extend (inheritance jaisa)
```ts
interface Person {
  name: string;
}
interface Employee extends Person {
  employeeId: number;
}
```

---

## 14. `type` vs `interface`

JS developer ke liye yeh confusion common hai — dono se "shape" define hoti hai, par farak hai:

```ts
// Type alias
type User = {
  name: string;
  age: number;
};

// Interface
interface User2 {
  name: string;
  age: number;
}
```

| Feature | `interface` | `type` |
|---|---|---|
| Object shape define | ✅ | ✅ |
| Union/Intersection | ❌ direct nahi | ✅ (`string \| number`) |
| Extend / Inherit | ✅ `extends` | ✅ `&` (intersection) |
| Re-open/merge same name | ✅ (declaration merging) | ❌ (duplicate error) |
| Primitives ko alias dena | ❌ | ✅ (`type ID = string`) |

**Rule of thumb:** Object/class shape define karna ho → `interface` use karo (especially React props). Union, intersection, ya primitive type ka alias chahiye ho → `type` use karo. Dono kaam karte hain, team convention follow karo.

---

## 15. Classes in TS

JS classes (`class`, `constructor`, `extends`) same hi hain. TS extra deta hai: **access modifiers, types, abstract classes, interfaces ka implementation.**

```ts
class Person {
  public name: string;       // bahar se accessible (default bhi yahi hai)
  private age: number;       // sirf isi class ke andar accessible
  protected city: string;    // class + subclass mein accessible
  readonly id: number;       // ek baar set, fir change nahi

  constructor(name: string, age: number, city: string, id: number) {
    this.name = name;
    this.age = age;
    this.city = city;
    this.id = id;
  }

  getAge(): number {
    return this.age;
  }
}

const p = new Person("Anand", 21, "Ranchi", 1);
console.log(p.name);     // ✅
console.log(p.age);      // ❌ error — private hai
```

### Shorthand constructor (bohot common pattern)
```ts
class Person {
  constructor(
    public name: string,
    private age: number
  ) {}
}
```
Yeh upar wale jaisa hi hai — TS automatically property bana deta hai, separate `this.name = name` likhne ki zaroorat nahi.

### Interface implement karna
```ts
interface Greetable {
  greet(): string;
}

class Person implements Greetable {
  constructor(public name: string) {}
  greet() {
    return `Hello, ${this.name}`;
  }
}
```

### Abstract class — blueprint jo directly instantiate nahi ho sakta
```ts
abstract class Shape {
  abstract area(): number;     // implementation child class mein zaroori
  describe(): string {
    return `Area is ${this.area()}`;
  }
}

class Circle extends Shape {
  constructor(private radius: number) { super(); }
  area(): number {
    return Math.PI * this.radius ** 2;
  }
}

new Shape();    // ❌ error — abstract class instantiate nahi ho sakta
new Circle(5);  // ✅
```

---

## 16. Generics — JS mein nahi hota, par bohot powerful

**Definition:** Generics ka matlab hai — function/interface/class likhte waqt type ko **fix nahi karna**, balki use karte waqt type **pass karna**. Ek tarah ka "type-level parameter".

Problem without generics:
```ts
function identity(value: any): any {
  return value;
}
let x = identity(5);
x.toUpperCase(); // ❌ no error dikhega abhi, par runtime crash karega — kyunki 'any' ne type-safety hata di
```

### Generic function
```ts
function identity<T>(value: T): T {
  return value;
}

let a = identity<number>(5);       // T = number
let b = identity<string>("hello"); // T = string
let c = identity(true);            // TS khud infer kar lega T = boolean
```
`<T>` ek placeholder hai jo call-time pe actual type se replace ho jaata hai. Naam `T` convention hai (Type), tum kuch bhi naam de sakte ho.

### Generic Interface
```ts
interface ApiResponse<T> {
  data: T;
  success: boolean;
}

const userResponse: ApiResponse<User> = {
  data: { name: "Anand", age: 21 },
  success: true,
};

const postsResponse: ApiResponse<string[]> = {
  data: ["post1", "post2"],
  success: true,
};
```
Yeh exactly wahi pattern hai jo tumhare GitHub analyzer project mein API responses ke liye kaam aayega — same `ApiResponse` shape, different `data` type.

### Generic Constraints (`extends`)
```ts
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}
getLength("hello");      // ✅ string has length
getLength([1, 2, 3]);    // ✅ array has length
getLength(5);             // ❌ number has no length
```
`extends` yahan "type must have at least these properties" ka matlab hai — class inheritance wala `extends` nahi hai.

### Generic Class
```ts
class Box<T> {
  constructor(private content: T) {}
  getContent(): T {
    return this.content;
  }
}

const numberBox = new Box<number>(123);
const stringBox = new Box<string>("hello");
```

---

## 17. Type Narrowing

Union type ke saath kaam karte waqt TS ko batana padta hai "is exact moment pe variable kaunsa specific type hai" — isi process ko **narrowing** kehte hain.

### `typeof` narrowing
```ts
function format(value: string | number) {
  if (typeof value === "string") {
    return value.trim();     // string ke methods available
  }
  return value.toFixed(2);   // number ke methods available
}
```

### `instanceof` narrowing
```ts
class Dog { bark() { console.log("Woof"); } }
class Cat { meow() { console.log("Meow"); } }

function makeSound(animal: Dog | Cat) {
  if (animal instanceof Dog) {
    animal.bark();
  } else {
    animal.meow();
  }
}
```

### `in` narrowing (object key check)
```ts
type Cat = { meow: () => void };
type Dog = { bark: () => void };

function makeSound(animal: Cat | Dog) {
  if ("meow" in animal) {
    animal.meow();
  } else {
    animal.bark();
  }
}
```

### Discriminated Union — production-grade pattern
Har type mein ek common "tag" property rakho (jaise `type` ya `kind`), fir us tag pe switch karo:

```ts
type SuccessResponse = { status: "success"; data: string };
type ErrorResponse = { status: "error"; message: string };
type ApiResult = SuccessResponse | ErrorResponse;

function handleResponse(res: ApiResult) {
  switch (res.status) {
    case "success":
      console.log(res.data);     // TS jaanta hai yahan data exist karta hai
      break;
    case "error":
      console.log(res.message);  // aur yahan message
      break;
  }
}
```
Yeh pattern tumhare AI interview-prep tool mein recruiter-scoring response ya roast-mode response handle karne ke liye perfect fit hoga — `{ mode: "roast", ... } | { mode: "normal", ... }`.

---

## 18. Utility Types

TS built-in mein kuch ready-made "type transformers" deta hai jo existing types se naya type banate hain. JS mein iska koi equivalent nahi hai.

```ts
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}
```

| Utility | Kaam | Example |
|---|---|---|
| `Partial<T>` | Saari properties optional bana deta hai | `Partial<User>` → update-form ke liye |
| `Required<T>` | Saari properties mandatory bana deta hai | `Required<User>` |
| `Pick<T, K>` | Sirf chuni hui keys rakhta hai | `Pick<User, "id" \| "name">` |
| `Omit<T, K>` | Chuni hui keys hata deta hai | `Omit<User, "password">` |
| `Readonly<T>` | Saari properties readonly | `Readonly<User>` |
| `Record<K, V>` | Key-value map type banata hai | `Record<string, number>` |

```ts
type PublicUser = Omit<User, "password">;
// { id: number; name: string; email: string }  -- password chala gaya

type UpdateUser = Partial<User>;
// saari fields optional ho gayi -- patch request ke liye perfect

type UserSummary = Pick<User, "id" | "name">;
// { id: number; name: string }

const scoreBoard: Record<string, number> = {
  Anand: 95,
  Rahul: 88,
};
```

**Real use-case (tumhare project ke liye):** Login response mein password kabhi frontend ko nahi bhejni — `Omit<User, "password">` se type-level guarantee mil jaati hai ki accidentally password field include nahi hogi.

---

## 19. Type Assertion

Kabhi-kabhi tumhe pata hota hai value ka type TS se zyada accurately, aur tum TS ko force karte ho "trust me, yeh type hai":

```ts
let value: unknown = "hello world";
let strLength: number = (value as string).length;

// alternate syntax (sirf .ts files mein, .tsx mein conflict karta hai)
let strLength2: number = (<string>value).length;
```

**Important:** Yeh type-conversion nahi hai (jaise `Number("5")`), yeh sirf compiler ko bolna hai "is type ko maan lo" — runtime pe kuch change nahi hota. Galat assertion karoge toh runtime crash ho sakta hai, isliye sparingly use karo.

```ts
const input = document.getElementById("username") as HTMLInputElement;
console.log(input.value); // bina assertion ke TS ko 'value' property pata nahi chalti
```

---

## 20. Modules

`import`/`export` syntax bilkul JS jaisa hi hai:

```ts
// math.ts
export function add(a: number, b: number): number {
  return a + b;
}
export interface Point {
  x: number;
  y: number;
}

// app.ts
import { add, Point } from "./math";
```

**Type-only import** (sirf type chahiye, runtime code nahi) — bundle size optimize karta hai:
```ts
import type { Point } from "./math";
```

---

## 21. TypeScript + React — Practical (Sketch AI jaisa project)

### Functional Component Props
```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: "primary" | "secondary"; // literal union
}

function Button({ label, onClick, variant = "primary" }: ButtonProps) {
  return <button className={variant} onClick={onClick}>{label}</button>;
}
```

### `useState` ke saath type
```tsx
const [count, setCount] = useState<number>(0);
const [user, setUser] = useState<User | null>(null);  // common pattern -- shuru mein null
```

### `useRef`
```tsx
const inputRef = useRef<HTMLInputElement>(null);
```

### Event handlers
```tsx
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  console.log(e.target.value);
}

function handleClick(e: React.MouseEvent<HTMLButtonElement>) {
  console.log("clicked");
}
```

### Redux Toolkit + TS (tumhare Affectra project jaisa)
```ts
interface EmotionState {
  current: string;
  history: string[];
}

const initialState: EmotionState = {
  current: "neutral",
  history: [],
};

const emotionSlice = createSlice({
  name: "emotion",
  initialState,
  reducers: {
    setEmotion: (state, action: PayloadAction<string>) => {
      state.current = action.payload;
      state.history.push(action.payload);
    },
  },
});
```
`PayloadAction<string>` Redux Toolkit ka generic type hai — batata hai dispatch hone wale action ka `payload` kis type ka hoga.

### Socket.io + TS
```ts
interface ServerToClientEvents {
  message: (text: string) => void;
}
interface ClientToServerEvents {
  sendMessage: (text: string) => void;
}

const socket: Socket<ServerToClientEvents, ClientToServerEvents> = io("http://localhost:5000");
socket.emit("sendMessage", "hello"); // type-checked event names + payload
```

---

## 22. TypeScript + Node/Express — Practical (Backend wala)

```bash
npm install express
npm install --save-dev typescript @types/node @types/express ts-node-dev
```
`@types/*` packages — yeh wo libraries hain jo khud TS mein nahi likhi gayi, par community ne unke liye type-definitions bana di hain (DefinitelyTyped project). Isse `express`, `node` jaisi plain-JS libraries bhi TS mein type-safe ban jaati hain.

```ts
import express, { Request, Response, NextFunction } from "express";

const app = express();

interface AuthRequest extends Request {
  user?: { id: string; role: string };
}

app.get("/profile", (req: AuthRequest, res: Response) => {
  res.json({ user: req.user });
});

function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  res.status(500).json({ message: err.message });
}
app.use(errorHandler);
```

### Mongoose + TS (MongoDB schema typing)
```ts
import { Schema, model, Document } from "mongoose";

interface IUser extends Document {
  name: string;
  email: string;
  password: string;
}

const userSchema = new Schema<IUser>({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
});

export const User = model<IUser>("User", userSchema);
```

---

## 23. `tsconfig.json` Deep Dive

Important options jo har project mein dikhenge:

```json
{
  "compilerOptions": {
    "target": "ES2020",          // output JS kaunsi JS version ka ho
    "module": "commonjs",         // ya "ESNext" agar ESM use kar rahe ho
    "strict": true,               // sabse important -- saare strict checks ek saath on
    "esModuleInterop": true,      // CommonJS modules ko import karne mein aasani
    "skipLibCheck": true,         // node_modules ke type-check skip (compile fast)
    "outDir": "./dist",           // compiled JS kahan jaaye
    "rootDir": "./src",           // source files kahan se aayein
    "jsx": "react-jsx",           // React projects ke liye zaroori
    "moduleResolution": "node",
    "resolveJsonModule": true
  },
  "include": ["src/**/*"]
}
```

`"strict": true` actually 8 alag flags ko ek saath on karta hai — sabse important hai `strictNullChecks` (null/undefined ko explicitly handle karna padega) aur `noImplicitAny` (har jagah type likhna zaroori, "any" silently allow nahi hoga). **Naye project mein hamesha `strict: true` rakho** — shuru mein thoda tedious lagega par bugs bohot kam hoge.

---

## 24. Best Practices + Common Mistakes

✅ **Karo:**
- Function params/return types explicitly likho, internal variables ke liye inference pe trust karo.
- `unknown` use karo `any` ki jagah jab type pata na ho.
- Discriminated unions use karo multiple "modes/states" handle karne ke liye (success/error, loading/loaded).
- `strict: true` always on rakho.
- Interfaces use karo React props/API shapes ke liye, type aliases unions/intersections ke liye.

❌ **Mat karo:**
- Har jagah `any` daal kar errors "fix" mat karo — yeh TS ka purpose hi khatam kar deta hai.
- Type assertion (`as`) ko overuse mat karo — yeh runtime check nahi karta, sirf compiler ko convince karta hai.
- Enum vs union literal confusion — naye React/Node projects mein union literal zyada lightweight hota hai.

---

## 25. Mini Practice Project + Exercises

**Project idea:** Ek `TaskManager` system bana jisme yeh sab concepts use ho:

```ts
type Priority = "low" | "medium" | "high"; // literal union

interface Task {
  id: number;
  title: string;
  completed: boolean;
  priority: Priority;
}

type TaskInput = Omit<Task, "id" | "completed">; // utility type
type TaskUpdate = Partial<Task>;                  // utility type

class TaskManager {
  private tasks: Task[] = [];
  private nextId = 1;

  addTask(input: TaskInput): Task {
    const task: Task = { id: this.nextId++, completed: false, ...input };
    this.tasks.push(task);
    return task;
  }

  updateTask(id: number, update: TaskUpdate): Task | undefined {
    const task = this.tasks.find((t) => t.id === id);
    if (!task) return undefined;
    Object.assign(task, update);
    return task;
  }

  getByPriority(priority: Priority): Task[] {
    return this.tasks.filter((t) => t.priority === priority);
  }
}

const manager = new TaskManager();
manager.addTask({ title: "Learn Generics", priority: "high" });
```

### Exercises (khud try karo)
1. `Cat | Dog | Bird` discriminated union banao, har ek ka alag `sound()` method ho.
2. `ApiResponse<T>` generic interface banao jisme `loading`, `error`, `data` teeno states handle ho.
3. `Role` ke liye string enum banao (`Admin`, `Editor`, `Viewer`) aur ek function jo role ke hisaab se permission return kare.
4. `User` interface banao, fir `Pick`, `Omit`, `Partial` teeno utility types use kar ke 3 alag variants banao.
5. Ek generic `Stack<T>` class banao with `push()`, `pop()`, `peek()` methods.

---

### Recap — sabse zyada use hone wale concepts (priority order)
1. Basic type annotations (variable/function)
2. Interfaces (React props, API shapes)
3. Union types + narrowing
4. Generics (reusable components, API responses)
5. Utility types (`Partial`, `Omit`, `Pick`)
6. Enums / literal unions (fixed-choice values)

Bas itna solid ho gaya toh tum kisi bhi MERN + TS project (jaise tumhara GitHub analyzer ya Sketch AI) ko TS mein migrate karne ke liye fully ready ho. 🚀
