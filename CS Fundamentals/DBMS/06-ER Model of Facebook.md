## 1. Recap — The 3 Steps to Build an ER Diagram

The same three steps from the previous lecture are reused here:

1. **Identify Entity Sets**
2. **Identify Relationships** (and mapping cardinalities)
3. **Identify Constraints** (participation constraints, etc.)

We will design the DB of a basic social-media network (Facebook) using the **ER Model**, producing an **ER Diagram**.

```
  The 3-step process applied to Facebook:

  Step 1: What are the "things" (entities)?
          --> User Profile, User Post, Post Comment, Post Like

  Step 2: How do they relate, and in what ratio?
          --> Posts (1:N), Friendship (M:N), Likes (1:N), etc.

  Step 3: Is participation total (mandatory) or partial (optional)?
          --> Every post MUST belong to a user (total on Post side), etc.
          |
          v
      ER Diagram (Conceptual Schema)
```

---

## 2. Features & Use Cases (Scope)

We model only **basic Facebook functionality** (not pages, reels, ads, etc.):

- **Profiles (User Profiles)** — anyone who signs up becomes a user.
- **Friends** — a user has friends, and those friends are themselves users (this needs a **unary / self relationship**).
- **Posts** — a user can post. A post contains text content, images, videos, etc.
- **Likes** — a post can be liked.
- **Comments** — a post can be commented on.
- **Timeline** — friends can visit your timeline and see all your posts. (This is a _retrieval feature_ implemented via DB queries: go to a profile → retrieve all posts associated with that profile → display on the timeline.)

---

## 3. Step 1 — Identify Entity Sets

Four entities are identified:

|#|Entity|Notes|
|---|---|---|
|1|**User Profile**|The signed-up user|
|2|**User Post**|A post made by a user|
|3|**Post Comment**|Comments on a post|
|4|**Post Like**|Likes on a post|

### Why Likes & Comments are SEPARATE entities (not attributes)

- A single post has **many likes** and **many comments**, each made by **many different users**.
- If we made _likes_ an **attribute** of Post, establishing a relationship becomes hard — because in the ER model, **a relationship is shown between two entities, NOT between an attribute and an entity.**
- Therefore Post-Like and Post-Comment are broken out into their **own separate entities**, so relationships with User Profile can be drawn.

```
  The problem with making Like an attribute of Post:

  User Profile ----[Can Like]----> ??? 
                                   You cannot draw this arrow to an attribute.
                                   In ER, relationships connect ENTITIES only.
                                   An attribute like "Like_Count" on Post tells
                                   you HOW MANY likes, but not WHO liked it or
                                   WHEN — and you cannot relate it to User Profile.

  The fix — make Post Like its own entity:

  User Profile ----[Can Like]----> Post Like <----[Has]---- User Post
       (1 user likes N post-likes)              (1 post has N post-likes)

  Now "Can Like" is a relationship between two entities:
  User Profile and Post Like. This is valid ER.
```

> **Design tip from the video:** While building a model, you often discover wrong assumptions mid-way. In the real world you go back and **refine the DB schema**. This clarity comes with practice.

---

## 4. Step 2 — Attribute Types of Each Entity

### Entity 1: User Profile

|Attribute|Type|
|---|---|
|User Name|**Primary Key**|
|Name|**Composite** attribute|
|Password|simple|
|Contact Number|**Multi-valued**|
|Email|**Multi-valued** (Facebook stores multiple emails)|
|Date of Birth|simple|
|Age|**Derived** (derived from DOB)|

```
  ER diagram notation reminder for attribute types:

  Oval with underline        --> Primary Key attribute  (User Name)
  Oval with sub-ovals        --> Composite attribute    (Name: First + Last)
  Double oval (concentric)   --> Multi-valued attribute (Contact Number, Email)
  Dashed oval                --> Derived attribute      (Age, computed from DOB)
  Plain oval                 --> Simple attribute       (Password, DOB)
```

### Entity 2: User Post

|Attribute|Type|
|---|---|
|Post ID|**Primary Key** (unique identification)|
|Content (text)|simple|
|Images|**Multi-valued** (a post can have multiple images)|
|Videos|**Multi-valued**|
|Created Time Stamp|simple|
|Modified Time Stamp|simple (Facebook later added the _edit_ feature)|

### Entity 3: Post Comment

|Attribute|Type|
|---|---|
|Post Comment ID|**Primary Key**|
|Text Content|simple|
|Time Stamp|simple|

### Entity 4: Post Like

- Has a **Post Like ID** as its primary key plus a time stamp.

---

## 5. Step 3 — Relationships, Cardinality & Constraints

Six relationships are established:

|#|Relationship|Between|Mapping Cardinality|Participation Constraint|
|---|---|---|---|---|
|1|**Friendship**|User Profile ↔ User Profile (unary/self)|**M : N**|—|
|2|**Posts**|User Profile → User Post|**1 : N**|**Total** on User Post (no orphan post; every post belongs to some user)|
|3|**Likes (Can Like)**|User Profile → Post Like|**1 : N**|**Total** on Post Like|
|4|**Comments**|User Profile → Post Comment|**1 : N**|**Total** on Post Comment|
|5|**Has**|User Post → Post Comment|**1 : N**|**Total** on Post Comment|
|6|**Has**|User Post → Post Like|**1 : N**|**Total** on Post Like|

### Reasoning given in the video

- **Friendship (M:N):** One user profile can be a friend of many profiles, and one friend can be a friend of many profiles → many-to-many. Since both sides are the same entity (User Profile), this is a **unary/self relationship**.

```
  Unary (Self) Relationship — Friendship:

  Both sides of the relationship are the same entity type (User Profile).
  This is drawn as a loop — the diamond connects back to the same rectangle.

        +-------------------+
        |   User Profile    |
        +-------------------+
                |
         [Friendship] (M:N)
                |
        +-------------------+
        |   User Profile    |  <-- same entity
        +-------------------+

  In ER notation this is drawn as a single rectangle with a
  relationship diamond looping back to itself.

  Why M:N? 
    Alice can be friends with Bob, Carol, Dave  (one user, many friends)
    Bob   can be friends with Alice, Eve, Frank (one friend, many users)
    --> Many-to-many.
```

- **Posts (1:N, total):** A post is always made by some user — "no orphan / _lavaaris_ post exists" → **total participation** on the post side.
- **Likes (1:N):** One user profile can like N posts; but a single like was made by exactly one user. Every like belongs to some user → total participation.
- **Comments (1:N):** One user profile can make N comments; one comment is made by exactly one user profile → total participation.
- **User Post HAS Post Comment (1:N, total):** A post can have N comments; every comment belongs to some post.
- **User Post HAS Post Like (1:N, total):** A post can have N likes; every like is associated with some post.

```
  Total participation — what it means:

  In ER diagrams, total participation is drawn with a DOUBLE LINE
  between the entity and the relationship diamond.

  Partial participation (single line): an entity MAY or MAY NOT
    participate. Example: a User Profile may or may not have posts.

  Total participation (double line): every instance MUST participate.
    Example: every Post Like MUST be associated with a User Post
    (a "floating" like that belongs to no post makes no sense).

  Post Comment has total participation in BOTH:
    - "Has" with User Post  (every comment belongs to some post)
    - "Comments" with User Profile (every comment was made by some user)
```

---

## 6. Final ER Diagram (Conceptual Schema)

Putting it together produces the **conceptual schema (ER diagram)** of Facebook using the same 3 steps:

1. Entity sets identified (4 entities, with the 4 stated assumptions — kept basic, not complicated).
2. Attributes assigned (Name = composite, Contact = multi-valued, User Name = PK, Age = derived, Email = multi-valued).
3. Relationships (6) with cardinalities and total-participation constraints.

```
  Simplified ER Diagram — Facebook (entities and relationships only):

                    [Friendship]
                    (M:N, unary)
                    /           \
                   /             \
  +--------------+                +--------------+
  | User Profile |---[Posts]----->|  User Post   |
  |              |   (1:N,total  |              |
  +--------------+  on Post side)+--------------+
         |    |                       |        |
         |    |                    [Has]     [Has]
    [Can |    | [Comments]         (1:N,    (1:N,
    Like]|    |  (1:N, total        total    total
    (1:N,|    |  on Comment)        on      on Like)
    total|    |                   Comment)    |
    on   |    |                      |        |
    Like)|    v                      v        v
         |  +-------------+   +-------------+ +----------+
         |  | Post Comment|   | Post Comment| | Post Like|
         |  +-------------+   +-------------+ +----------+
         v
      +----------+
      | Post Like|
      +----------+

  Cleaner reading: Post Comment participates in TWO relationships:
    (a) "Comments" with User Profile (who wrote it)
    (b) "Has"      with User Post    (which post it belongs to)

  Likewise Post Like participates in TWO relationships:
    (a) "Can Like" with User Profile (who liked it)
    (b) "Has"      with User Post    (which post was liked)
```

> **Key takeaway:** Had we made _likes_ an attribute of User Post, we could not have established the like relationship. Deciding to model it as a separate entity is a skill that **comes only with practice** — by drawing and studying many ER diagrams.

**Next lecture:** Relational Model — and later, how to convert an ER diagram into a relational model.