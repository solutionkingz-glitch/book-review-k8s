In backend/src/server.js, the listen call was changed to bind explicitly to all interfaces:

    app.listen(PORT, "0.0.0.0", () => console.log(`Server running on port ${PORT}`));
