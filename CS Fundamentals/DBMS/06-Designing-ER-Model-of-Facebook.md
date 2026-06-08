## 1. Recap — The 3 Steps to Build an ER Diagram

1. **Identify Entity Sets**
2. **Identify Relationships** (and mapping cardinalities)
3. **Identify Constraints** (participation constraints, etc.)

Design the DB of a basic social-media network (Facebook) using the **ER Model**, producing an **ER Diagram**.

---
## 2. Features & Use Cases (Scope)

Model only **basic Facebook functionality** (not pages, reels, ads, etc.):

- **Profiles (User Profiles)** — anyone who signs up becomes a user.
- **Friends** — a user has friends, who are themselves users (needs a **unary / self relationship**).
- **Posts** — a user can post. A post contains text content, images, videos, etc.
- **Likes** — a post can be liked.
- **Comments** — a post can be commented on.
- **Timeline** — friends can visit your timeline and see all your posts. A *retrieval feature* implemented via DB queries: go to a profile → retrieve all posts associated with that profile → display on the timeline.

---
## 3. Step 1 — Identify Entity Sets

| # | Entity | Notes |
|---|--------|-------|
| 1 | **User Profile** | The signed-up user |
| 2 | **User Post** | A post made by a user |
| 3 | **Post Comment** | Comments on a post |
| 4 | **Post Like** | Likes on a post |

### Why Likes & Comments are SEPARATE entities (not attributes)

- A single post has **many likes** and **many comments**, each made by **many different users**.
- If *likes* were an **attribute** of Post, establishing a relationship becomes hard — in the ER model, **a relationship is shown between two entities, NOT between an attribute and an entity.**
- Therefore Post-Like and Post-Comment are broken out into their **own separate entities**, so relationships with User Profile can be drawn.

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

> **Design tip:** While building a model, you often discover wrong assumptions mid-way. You go back and **refine the DB schema**. This clarity comes with practice.

---

## 4. Step 2 — Attribute Types of Each Entity

### Entity 1: User Profile
| Attribute | Type |
|-----------|------|
| User Name | **Primary Key** |
| Name | **Composite** attribute |
| Password | simple |
| Contact Number | **Multi-valued** |
| Email | **Multi-valued** (Facebook stores multiple emails) |
| Date of Birth | simple |
| Age | **Derived** (derived from DOB) |

```
  ER diagram notation reminder for attribute types:

  Oval with underline        --> Primary Key attribute  (User Name)
  Oval with sub-ovals        --> Composite attribute    (Name: First + Last)
  Double oval (concentric)   --> Multi-valued attribute (Contact Number, Email)
  Dashed oval                --> Derived attribute      (Age, computed from DOB)
  Plain oval                 --> Simple attribute       (Password, DOB)
```

### Entity 2: User Post
| Attribute | Type |
|-----------|------|
| Post ID | **Primary Key** (unique identification) |
| Content (text) | simple |
| Images | **Multi-valued** (a post can have multiple images) |
| Videos | **Multi-valued** |
| Created Time Stamp | simple |
| Modified Time Stamp | simple (the *edit* feature) |

### Entity 3: Post Comment
| Attribute | Type |
|-----------|------|
| Post Comment ID | **Primary Key** |
| Text Content | simple |
| Time Stamp | simple |

### Entity 4: Post Like
- Has a **Post Like ID** as its primary key plus a time stamp.

---

## 5. Step 3 — Relationships, Cardinality & Constraints

Six relationships are established:

| #   | Relationship         | Between                                  | Mapping Cardinality | Participation Constraint                                                 |
| --- | -------------------- | ---------------------------------------- | ------------------- | ------------------------------------------------------------------------ |
| 1   | **Friendship**       | User Profile ↔ User Profile (unary/self) | **M : N**           | —                                                                        |
| 2   | **Posts**            | User Profile → User Post                 | **1 : N**           | **Total** on User Post (no orphan post; every post belongs to some user) |
| 3   | **Likes (Can Like)** | User Profile → Post Like                 | **1 : N**           | **Total** on Post Like                                                   |
| 4   | **Comments**         | User Profile → Post Comment              | **1 : N**           | **Total** on Post Comment                                                |
| 5   | **Has**              | User Post → Post Comment                 | **1 : N**           | **Total** on Post Comment                                                |
| 6   | **Has**              | User Post → Post Like                    | **1 : N**           | **Total** on Post Like                                                   |

### Relationship Justification
#### 1. Friendship (M:N)
A user can have many friends, and a friend can be connected to many users. Since both participants are **User Profile**, this is a **unary (self) relationship**.
```
User Profile ── Friendship ── User Profile        (M)                     (N)
```
#### 2. Posts (1:N)
A user can create multiple posts, but each post is created by exactly one user.
- One User Profile → Many User Posts
- Every post must belong to a user → **Total participation on User Post**
#### 3. Likes (1:N)
A user can create many likes, but each like is made by exactly one user.
- One User Profile → Many Post Likes
- Every like must be associated with a user → **Total participation on Post Like**
#### 4. Comments (1:N)
A user can write many comments, but each comment is written by exactly one user.
- One User Profile → Many Post Comments
- Every comment must belong to a user → **Total participation on Post Comment**
#### 5. User Post HAS Post Comment (1:N)
A post can have multiple comments, but each comment belongs to exactly one post.
- One User Post → Many Post Comments
- Every comment must belong to a post → **Total participation on Post Comment**
#### 6. User Post HAS Post Like (1:N)
A post can receive multiple likes, but each like is associated with exactly one post.
- One User Post → Many Post Likes
- Every like must belong to a post → **Total participation on Post Like**
### Participation Constraints

- **Partial Participation:** Entity may or may not participate in a relationship.
    - Example: A user may have zero posts.
- **Total Participation:** Every entity instance must participate in the relationship.
    - Every Post, Like, and Comment must be associated with their corresponding User/Post.
    - Represented by a **double line** in ER diagrams.
---
## 6. Final ER Diagram (Conceptual Schema)
Putting it together produces the **conceptual schema (ER diagram)** of Facebook using the same 3 steps:

1. Entity sets identified (4 entities, 4 stated assumptions — kept basic).
2. Attributes assigned (Name = composite, Contact = multi-valued, User Name = PK, Age = derived, Email = multi-valued).
3. Relationships (6) with cardinalities and total-participation constraints.
 ![[Pasted image 20260606155546.png]]

> **Key takeaway:** Had *likes* been an attribute of User Post, the like relationship could not have been established. Modeling it as a separate entity is a skill that **comes only with practice** — by drawing and studying many ER diagrams.

**Next lecture:** Relational Model — and later, how to convert an ER diagram into a relational model.
