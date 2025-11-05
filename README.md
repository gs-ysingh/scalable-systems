# Scalable Systems

A collection of system design notes, diagrams, and architectural deep-dives for building large-scale distributed systems.

## 📚 System Design Case Studies

### Social Media & Feeds

- **[News Feed System Design](./feeds.md)** - Facebook/Twitter-style feeds, fan-out strategies, precomputed feeds, DynamoDB indexing

### Communication & Messaging Platforms

- **[WhatsApp System Design](./whatsapp-docs.md)** - Real-time messaging, offline messages, group chats, end-to-end encryption

### Collaboration Tools

- **[Google Docs System Design](./google-docs.md)** - Real-time collaborative editing, Operational Transform (OT), concurrent multi-user editing

### Storage & File Sharing

- **[Dropbox / Google Drive System Design](./google-drive.md)** - File upload/download, presigned URLs, chunking, resumable uploads, sync strategies

### Video Streaming

- **[YouTube System Design](./youtube.md)** - Video upload/streaming, adaptive bitrate, video processing pipeline, CDN distribution

### Location-Based Services

- **[Google Maps System Design](./google-maps-learnings.md)** - Routing algorithms, graph databases, real-time traffic updates with Kafka/Flink

### Search & Autocomplete

- **[Google Autocomplete System Design](./google-autocomplete.md)** - Search suggestions, autocomplete, ranking algorithms, caching strategies

### URL Management

- **[Bit.ly / URL Shortener System Design](./bit-ly.md)** - URL shortening, short code generation, read-heavy scaling, caching strategies, counter batching

### Web Crawling

- **[Web Crawler System Design](./webcrawler.md)** - Web crawling for search engines and LLM training, fault tolerance, politeness, distributed crawling

### E-commerce & Auctions

- **[Online Auction System Design](./online-auction.md)** - Bidding system, strong consistency (OCC), real-time updates (SSE), fault tolerance (Kafka), dealing with contention

---

## Notes

Examples use Indian geography (Delhi, Mumbai, Bengaluru, etc.) for better relatability, but architectural principles apply globally.

---

**Happy Learning!** 🎓
