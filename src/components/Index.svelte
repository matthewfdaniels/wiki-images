<script>
	import { getContext } from "svelte";
	import Footer from "$components/Footer.svelte";
	import GitHubContributors from "$components/GitHubContributors.svelte";

	// const copy = getContext("copy");
	// const data = getContext("data");
</script>

<svelte:boundary onerror={(e) => console.error(e)}>
	<GitHubContributors />
	<!-- <Footer recirc={true} /> -->
</svelte:boundary>




