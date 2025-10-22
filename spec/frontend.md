## **Project Specification: Dharma Stream**

### **Part 3: Frontend Specification (React)**

**Version:** 1.3
**Date:** October 21, 2025

---

### **1. Frontend Philosophy & Guiding Principles**

The Dharma Stream frontend is the primary interface for all users. Its design and implementation must prioritize clarity, serenity, and performance.

*   **Modern & Responsive:** The UI must be built mobile-first, providing an excellent experience on small touchscreens, and gracefully scaling up to large, high-resolution desktop monitors.
*   **Performant:** The application will be architected for speed. This includes fast initial loads, smooth transitions, and efficient data fetching. We will leverage code splitting, lazy loading, and intelligent caching.
*   **State-Driven:** The UI will be a direct reflection of the application's state. We will use robust state management libraries to ensure data consistency and eliminate UI bugs.
*   **Component-Based:** The application will be composed of small, reusable, and well-defined React components, promoting maintainability and testability.
*   **Secure by Design:** The frontend will never store sensitive information or contain business logic that should be on the backend. It will operate as a secure client to the Supabase API, relying on JWTs for authentication and RLS for data authorization.

### **2. Technology Stack & Libraries**

| Category | Library/Tool | Rationale |
| :--- | :--- | :--- |
| **Framework** | React 18+ with TypeScript | Industry standard for building interactive UIs with the benefits of static typing for robustness. |
| **Build Tool** | Vite | Provides a significantly faster development experience and optimized production builds compared to older tools. |
| **Routing** | React Router | The de-facto standard for handling client-side routing in React applications. |
| **Server State** | TanStack Query (React Query) | The best-in-class solution for fetching, caching, synchronizing, and updating server state. It eliminates boilerplate `useEffect` and `useState` for data fetching. |
| **Global Client State** | Zustand | A minimalist, fast, and scalable state management solution for global client-side state (e.g., auth status, UI settings). |
| **UI Components** | Shadcn/UI (or similar like MUI, Mantine) | A component library to accelerate development. Shadcn/UI is recommended for its accessibility, customizability, and "copy-and-paste" philosophy. |
| **Styling** | Tailwind CSS | A utility-first CSS framework that allows for rapid development of custom, responsive designs without writing custom CSS. |
| **Media Player** | ReactPlayer | A versatile and battle-tested component for playing a variety of media URLs, including HLS streams. |
| **Supabase Client** | `@supabase/supabase-js` | The official JavaScript library for interacting with the Supabase backend. |

### **3. Project Structure (`frontend/src/`)**

```
src/
├── api/
│   └── supabase.ts         # Supabase client initialization and typed data fetching functions.
├── assets/                 # Static assets like images, fonts.
├── components/
│   ├── layout/             # Main layout components (Navbar, Sidebar, Footer).
│   ├── ui/                 # Reusable, generic UI components (Button, Card, Dialog - from Shadcn/UI).
│   └── specific/           # Complex components specific to this app (MediaPlayer, SubtitleList, etc.).
├── hooks/
│   └── useAuth.ts          # Custom hook to access user and session data globally.
├── lib/
│   └── utils.ts            # Utility functions (e.g., date formatting).
├── pages/
│   ├── public/             # Routes accessible to all (LoginPage, AuthCallback).
│   ├── user/               # Protected routes for authenticated users (HomePage, PlayerPage).
│   └── admin/              # Protected routes for admins (UserManagement, SubtitleReview).
├── services/               # (Alternative to api/)
├── state/
│   └── authStore.ts        # Zustand store for managing authentication state.
├── styles/
│   └── globals.css         # Global styles and Tailwind CSS imports.
└── App.tsx                 # Main application component with routing setup.
└── main.tsx                # Application entry point.
```

### **4. Component Architecture & User Flows**

#### **4.1. Public & Authentication Flow**

*   **`LoginPage.tsx`:**
    *   **UI:** A simple, centered form with fields for email and password. Includes links for "Forgot Password" and "Sign Up".
    *   **Logic:** Uses the `supabase.auth.signInWithPassword()` function. Manages loading and error states. On success, Supabase handles the session and the user is redirected to the home page.
*   **`AuthListener` (in `App.tsx`):**
    *   **Logic:** A central component that subscribes to `supabase.auth.onAuthStateChange`.
    *   It updates the Zustand `authStore` with the user's session information whenever it changes (login, logout).
    *   This global store is then used by protected routes and hooks to determine the user's authentication status.

#### **4.2. Authenticated User Flow**

*   **`HomePage.tsx` (`/`):**
    *   **Data Fetching:** Uses two parallel `useQuery` hooks (from TanStack Query):
        1.  Fetches the user's 5 most recently viewed items from `user_view_history` (joining `media_files`).
        2.  Fetches the categories assigned to the user from `user_categories` (joining `categories`).
    *   **UI:**
        *   Displays a "Resume Watching" section with `MediaCard` components for the history items.
        *   Displays a "Browse Categories" section with `CategoryCard` components.
        *   Handles loading and empty states gracefully.
*   **`PlayerPage.tsx` (`/media/:mediaId`):**
    *   **Data Fetching:**
        *   `useQuery` to fetch all metadata for the `mediaId` from the URL.
        *   `useQuery` to fetch all subtitles for the `mediaId`.
        *   `useQuery` to fetch the user's view history for this specific `mediaId`.
    *   **UI:**
        *   A main `MediaPlayer` component at the top.
        *   A `SubtitleList` component below the player.
    *   **Logic:**
        *   Constructs the media URL (`.m3u8` for video, `.m4a` for audio) based on the fetched metadata and user preference.
        *   Passes the subtitles and history data to the child components.
*   **`MediaPlayer.tsx` (Specific Component):**
    *   **Props:** `mediaUrl`, `initialSeekTime`, `onProgressUpdate`, `onEnded`.
    *   **UI:** Wraps `ReactPlayer`. Includes a custom UI toggle for "Video" / "Audio Only".
    *   **Logic:**
        *   On mount, seeks to `initialSeekTime`.
        *   Saves the user's video/audio preference to `localStorage`.
        *   Calls the `onProgressUpdate` prop function every ~15 seconds, passing the current playback time. This function is debounced to prevent excessive API calls.
        *   Calls the `onEnded` prop when playback finishes.
*   **`SubtitleList.tsx` (Specific Component):**
    *   **Props:** `subtitles`, `currentTime`.
    *   **UI:**
        *   Renders a virtualized list of subtitle segments for performance.
        *   Each segment displays the text (`fixed_text` if available, otherwise `text`).
        *   The segment whose `startTime` and `endTime` includes the `currentTime` prop is highlighted.
        *   A "Fix" button is shown *only* on the highlighted segment.
    *   **Logic:**
        *   Clicking "Fix" opens the `CorrectionDialog`, passing the `subtitle` object.
*   **`CorrectionDialog.tsx` (UI Component):**
    *   **UI:** A modal dialog with a text area pre-filled with the current text. Displays AI suggestions as clickable buttons.
    *   **Logic:** On submit, it calls a `useMutation` hook (from TanStack Query) to insert a new record into the `user_corrections` table. It handles loading/success/error states and closes the dialog on success.

#### **4.3. Admin Flow (`/manage/*`)**

This section is protected by a route guard that checks if the user has the `admin` role from their JWT.

*   **`UserManagementPage.tsx` (`/manage/users`):**
    *   **UI:** A two-panel layout as described in the overview.
    *   **Data Fetching:**
        *   `useQuery` to fetch all users.
        *   When a user is selected, a `useQuery` fetches all categories, and another `useQuery` fetches the categories specifically assigned to that user.
    *   **Logic:**
        *   Manages the state of the checkboxes.
        *   On "Save", a `useMutation` hook sends the list of selected `category_ids` to the custom Python API endpoint.
*   **`SubtitleReviewPage.tsx` (`/manage/review`):**
    *   **UI:** A list of `CorrectionReviewCard` components and a single "Confirm & Approve" button at the bottom.
    *   **Data Fetching:** `useQuery` to fetch the 100 most recent pending corrections.
    *   **Client-Side State:** Uses `useState` to hold the array of corrections, each with a `localStatus` field.
    *   **Logic:**
        *   The "Reject" button on each card updates the `localStatus` in the component's state, causing a re-render.
        *   The "Confirm & Approve" button filters the state array into `approved_ids` and `rejected_ids`, then calls a `useMutation` hook to send this data to the bulk processing API endpoint. On success, it invalidates the initial `useQuery` to refetch the list.

### **5. State Management Strategy**

*   **Server Cache (TanStack Query):** The single source of truth for any data that comes from the server. It handles caching, background refetching, and optimistic updates. All interactions with the Supabase database will be wrapped in `useQuery` (for reading) and `useMutation` (for writing).
*   **Global UI State (Zustand):** Used for a small set of truly global client-side state that does not persist on the server.
    *   **`authStore`:** Stores the current user session object. This is the source of truth for authentication status across the app.
    *   **`uiStore` (Optional):** Could manage things like theme (light/dark mode), or whether a global sidebar is open or closed.
*   **Local Component State (`useState`):** Used for state that is local to a single component and its children, such as form input values, modal open/closed state, or the `localStatus` of corrections in the admin review page.

### **6. API Interaction (`api/supabase.ts`)**

This file will centralize all communication with Supabase.

```typescript
// api/supabase.ts
import { createClient } from '@supabase/supabase-js';
// ... import database types generated by Supabase CLI

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

export const supabase = createClient<Database>(supabaseUrl, supabaseAnonKey);

// Example of a typed fetching function
export const getMediaDetails = async (mediaId: string) => {
  const { data, error } = await supabase
    .from('media_files')
    .select('*')
    .eq('id', mediaId)
    .single();

  if (error) throw error;
  return data;
};
```
Components will then use these functions within `useQuery`:
`const { data: media } = useQuery(['media', mediaId], () => getMediaDetails(mediaId));`
