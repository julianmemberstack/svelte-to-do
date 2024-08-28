<script>
    import "../app.css";
</script>

<slot />

<style>
    :global(body) {
        font-family: 'DM Sans', sans-serif;
    }

    /* You can also set specific weights if needed */
    :global(h1, h2, h3, h4, h5, h6) {
        font-weight: 700; /* Bold */
    }

    :global(button, input) {
        font-family: 'DM Sans', sans-serif;
    }
</style>