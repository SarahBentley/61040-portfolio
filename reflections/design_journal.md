# Design Decisions in First Iteration (LLM Summary)
Here’s the design we converged on for Common Understandings.
- The core product goal is to help researchers build understanding over time, not manage papers or maintain a knowledge graph.
- The key workflow is: capture → retrieve → synthesize → share.
- The system should minimize persistent organization. Users should not have to maintain folders, tags, or manual graph structure.
- AI should help with semantic retrieval and surfacing relevant prior material, but the user should still do the actual synthesis and thinking.
The main domain distinction is now:
- Papers are sources
- Understandings are the durable user-created artifacts
- A paper may be saved without an understanding yet.
- An understanding can be based on one paper, multiple papers, prior understandings, or a mix.
- Users do not save other users’ understandings as a primary action. Instead, they can create a new understanding from them, discuss them, or use them as references.
- Saved papers do not get their own prominent “Saved Papers” section in the profile/library, because that would make the product feel like a paper manager again.
The main UI has three core surfaces:
1. Feed
   - Social-media-like posts from lab members.
   - Each post contains a user’s commentary/understanding plus a paper.
   - Paper actions include:
     - Save paper
     - Create understanding
   - Comments/discussion appear beneath or expand like a normal social feed.
2. My Understandings / Profile
   - Main content is a collection of the user’s understandings.
   - Examples can be questions, themes, literature reviews, or things like “Papers that changed my viewpoint on X.”
   - A persistent chat/search panel lets the user semantically search across their understandings and saved papers.
3. Understanding workspace
   - Main area is a graph-paper-style canvas.
   - It contains cards/stickies for source papers, prior understandings, themes, and notes.
   - The sidebar has a contextual chat.
   - Referenced papers and prior understandings are visible directly in the sidebar, so the user can inspect them without opening lots of separate windows.
   - The user is doing the synthesis; AI mainly helps retrieve and surface relevant material.
The concept set became:
- Saving [User, Paper]
  - Purpose: preserve papers for later use without immediate organization.
  - Saving is specifically for papers now, not arbitrary items.
  - It is lightweight and should not automatically create a blank understanding.
- Composing [Author, Item]
  - Purpose: let authors build a new artifact by combining and revising existing material.
  - In this app, a Composition is an Understanding.
  - Item can be a paper or a prior understanding.
  - create should take:
    - author
    - content
    - items
  - It should not force blank initial content or empty initial items.
  - addItem, removeItem, edit, and remove let the understanding evolve later.
- SemanticSearching [Item]
  - Purpose: retrieve items by semantic similarity rather than exact wording or predefined organization.
  - We decided to explicitly model embeddings.
  - Each searchable item has its own embedding.
  - A paper’s embedding comes from paper text.
  - An understanding’s embedding comes from the understanding’s written content.
  - The paper references inside an understanding are not re-embedded as part of the understanding.
  - Composing stores provenance/relationships; SemanticSearching stores semantic representations.
- Posting [Author, Item, Forum]
  - Used to share understandings to a lab/group forum.
- Commenting [Author, Target]
  - Used to discuss posts.
  - The comment target is a Post.
  - Replies are represented through a parent Comment relation, rather than treating Comment itself as another generic Target.
The semantic-search mental model became:
Paper P1         -> embedding(paper text)
Paper P2         -> embedding(paper text)
Understanding U1 -> embedding(user-written understanding text)

And separately:
Understanding U1 includes {P1, P2}

The relationship belongs to Composing; the vectors belong to SemanticSearching.
For reactions, the important decisions were:
- Creating an understanding should make that understanding searchable.
- Editing an understanding should update its semantic representation.
- Saving a paper should save the paper and make the paper searchable.
- If a brand-new paper is introduced while creating an understanding, the application-level request should also save and index that paper.
- Deleting an understanding should remove it from semantic search.
- Deleting an understanding should remove posts sharing it.
- Posting and commenting can be triggered through Requesting actions.
- Removing a post should remove comments attached to it.
The cleanest handling for new papers during composition was to make the request explicitly carry them, rather than making Composing responsible for ingestion:
Requesting.createUnderstanding(
  author,
  content,
  items,
  newPapers
)

where newPapers contains (paper, paperContent) pairs.
Then:
- items contains everything referenced by the new understanding.
- newPapers contains only papers that are entering the system for the first time.
- Those new papers get both Saving.save(...) and SemanticSearching.add(...).
- The new composition separately gets SemanticSearching.add(composition, content).
We also decided that “find all understandings/posts/comments related to a paper” is not a reaction. It is better thought of as a view/query assembled across concept states, because nothing is changing when the user asks for that view.
For the user journey, the canonical scenario became:
- researcher sees an interesting zeroth-order optimization paper in the feed
- saves it without needing to read/organize it
- months later starts “How to Optimize ML Models”
- semantic search resurfaces that saved paper plus prior understandings
- researcher reads the paper, writes a brief understanding, reviews related prior understandings
- uses the understanding workspace/canvas to synthesize state-of-the-art optimization methods
- the resulting synthesis is itself preserved as an understanding for future use
The overarching design principle we kept returning to is:
The user should preserve and build understanding, not maintain structure.

And the strongest product distinction is:
You collect understandings; papers support them.