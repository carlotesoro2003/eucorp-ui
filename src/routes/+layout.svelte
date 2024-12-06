<script lang="ts">
  import "../app.css";
  import "tailwindcss/tailwind.css";
  import { goto } from "$app/navigation";
  import { supabase } from "$lib/supabaseClient";
  import { onMount } from "svelte";
  import { page } from "$app/stores";
  import { userStore } from "$lib/stores/userStore";
  import Login from "$lib/components/auth/Login.svelte";
  import AppSidebar from "$lib/components/app-sidebar.svelte";
  import * as Breadcrumb from "$lib/components/ui/breadcrumb/index.js";
  import { Separator } from "$lib/components/ui/separator/index.js";
  import * as Sidebar from "$lib/components/ui/sidebar/index.js";
  import {ModeWatcher} from "mode-watcher";

  import Bell from "lucide-svelte/icons/bell";
  import Sun from "lucide-svelte/icons/sun";
  import Moon from "lucide-svelte/icons/moon";
 
  import { toggleMode } from "mode-watcher";
  import { Button } from "$lib/components/ui/button/index.js";


  let currentPath = "";
  let loading = true;
  let isDarkMode = true;

  $: currentPath = $page.url.pathname;

  // Subscribe to userStore
  let user: { session: any; isVerified: boolean; userRole: string | null; profilePic: string | null; email: string | null; departmentName: string } = {
    session: null,
    isVerified: false,
    userRole: null,
    profilePic: null,
    email: null,
    departmentName: "",
  };
  $: userStore.subscribe((value) => {
    user = value;
  });

  const ensureSession = async () => {
  if (!user.session) {
    loading = false;
    goto("/login");
    return;
  }

  try {
    const { session } = user;
    const { data: profileData, error: fetchError } = await supabase
      .from("profiles")
      .select("id, role, is_verified, profile_pic, department_id, email, first_name, last_name")
      .eq("id", session.user.id)
      .single();

    if (fetchError) {
      if (fetchError.code === "PGRST116") {
        const { error: insertError } = await supabase
          .from("profiles")
          .insert({
            id: session.user.id,
            email: session.user.email,
            first_name: session.user.user_metadata?.first_name || "New",
            last_name: session.user.user_metadata?.last_name || "User",
            role: "user",
            is_verified: false,
          });

        if (insertError) {
          goto("/login");
          return;
        }
      } else {
        goto("/");
        return;
      }
    }

    if (profileData) {
      userStore.set({
        session,
        isVerified: profileData.is_verified ?? false,
        userRole: profileData.role ?? null,
        profilePic: profileData.profile_pic,
        email: profileData.email,
        departmentName: profileData.department_id
          ? (
              await supabase
                .from("departments")
                .select("name")
                .eq("id", profileData.department_id)
                .single()
            ).data?.name ?? "No Department"
          : "",
        firstName: profileData.first_name || "New",
        lastName: profileData.last_name || "User",
      });

      if (!profileData.is_verified) {
        goto("/login");
        return;
      }
    }
  } catch (err) {
    goto("/login");
  } finally {
    loading = false;
  }
};

  async function logout() {
    const { error } = await supabase.auth.signOut();
    if (error) console.error("Error logging out:", error.message);
    userStore.set({
      session: null,
      isVerified: false,
      userRole: null,
      departmentName: "",
      profilePic: null,
      email: null,
      firstName: "New",
      lastName: "User",
    });
    goto("/login");
  }

  function toggleTheme() {
    isDarkMode = !isDarkMode;
    document.documentElement.setAttribute(
      "data-theme",
      isDarkMode ? "dark" : "light"
    );
  }

  onMount(() => {
    ensureSession();
    const { data: authListener } = supabase.auth.onAuthStateChange(
      (_, newSession) => {
        userStore.update((u) => ({
          ...u,
          session: newSession,
        }));
        if (!newSession) goto("/login");
        else ensureSession();
      }
    );
    return () => {
      authListener.subscription.unsubscribe();
    };
  });
</script>

<ModeWatcher />
<!-- Render loading spinner during session validation -->
{#if loading}
  <div class="flex items-center justify-center min-h-screen bg-base-300">
    <span class="loading loading-spinner loading-lg"></span>
  </div>

{:else if user.session && user.isVerified}
<Sidebar.Provider>
  <AppSidebar />
  <Sidebar.Inset>
    <header
      class="flex h-16 shrink-0 items-center justify-between gap-2 transition-[width,height] ease-linear group-has-[[data-collapsible=icon]]/sidebar-wrapper:h-12"
    >
      <!-- Left section -->
      <div class="flex items-center gap-2 px-4">
        <Sidebar.Trigger class="-ml-1" />
        <Separator orientation="vertical" class="mr-2 h-4" />
        <Breadcrumb.Root>
          <Breadcrumb.List>
            <Breadcrumb.Item class="hidden md:block">
              <Breadcrumb.Link href="#">Building Your Application</Breadcrumb.Link>
            </Breadcrumb.Item>
            <Breadcrumb.Separator class="hidden md:block" />
            <Breadcrumb.Item>
              <Breadcrumb.Page>Data Fetching</Breadcrumb.Page>
            </Breadcrumb.Item>
          </Breadcrumb.List>
        </Breadcrumb.Root>
      </div>

      <!-- Right section -->
      <div class="flex items-center gap-3 pr-4">
        <Button onclick={toggleMode} variant="outline" size="icon">
          <Sun
            class="h-[1.2rem] w-[1.2rem] rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0"
          />
          <Moon
            class="absolute h-[1.2rem] w-[1.2rem] rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100"
          />
          <span class="sr-only">Toggle theme</span>
        </Button>
        <Button variant="outline" size="icon" onclick={() => alert('Notifications clicked!')}>
          <Bell class="h-[1.2rem] w-[1.2rem]" />
          <span class="sr-only">Notifications</span>
        </Button>
      </div>
    </header>
    <div class="flex flex-1 flex-col gap-4 p-4 pt-0">
      <slot />
    </div>
  </Sidebar.Inset>
</Sidebar.Provider>


{:else if !user.session}
  <Login />

{:else}
  <div class="flex items-center justify-center min-h-screen bg-base-300">
    <div class="text-center">
      <h1 class="text-2xl font-bold text-yellow-500 mb-4">Pending for System Access</h1>
      <p class="text-white">Your account is under review. Please wait for verification.</p>
    </div>
  </div>
{/if}


