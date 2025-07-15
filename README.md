# [개인 프로젝트 - Django로 게시판 만들기]

#### 프로젝트 목차
- [01. Initial setting](/docs/01-Initial-setting.md)
- [02. Create Model](/docs/02-model.md)
- [03. Django Administrator](/docs/03-Django-administrator.md)
---
- [04. READ Posts, Post](/docs/04-READ-Post.md)
- [05. CREATE Post](/docs/05-CREATE-Post.md)
- [06. UPDATE Post](/docs/06-UPDATE-Post.md)
- [07. DELETE Post](/docs/07-DELETE-Post.md)
---
- [08. Django static, bootstrap](/docs/08-Static&Bootstrap.md)
- [09. Template inheritance](/docs/09-Template-inheritance.md)
- [10. Navigation bar](/docs/10-Navigation-bar.md)
- [11. Post list paging](/docs/11-Post-list-paging.md)
---
- [12. Log In & Log Out](/docs/12-Sign-in&Sign-out.md)
- [13. Sign Up](/docs/13-Sign-Up.md)
- [14. Add author to Post](/docs/14-Add-author-to-Post.md)
- [15. Only the author can UPDATE, DELETE](/docs/15-Only-the-author-can-UPDATE-DELETE-posts.md)
---
- [16. URL routing structure (Flat, Nested, shallow)](/docs/16-URL-routing-structure.md)
- [17. CREATE, READ Comment (SSR) + number of comments](/docs/17-CREATE-READ-Commemt(SSR).md)
- [18. Separate 'views.py' by function](/docs/18-Separate-views-by-function.md)
- [19. READ Comment (Async JsonResponse)](/docs/19-READ-Comments-(Async-JsonResponse).md)
- [20. CREATE Comment (Async)](/docs/20-CREATE-Comment-(Async).md)
- [21. UPDATE Comment (Async)](/docs/21-UPDATE-Comment-(Async).md)
- [22. DELETE Comment (Async)](/docs/22-DELETE-Comment-(Async).md)
- [23. Async login & Store draft data](/docs/23-Async-login&Store-draft-data.md)
- [23. Comments paging]
---
- [24. Vote function (Async)]
- [25. Search post in post_list (basic 'title' search)]
- [26. Search post in post_list (input 'search_target')]
---
- [27. Password reset (with using Django PasswordResetView)]
- [28. Password reset (secure secret keys with using django-environ)]
- [29. Password change]
---
- [30. Member's profile page(show created posts, comments, voted_posts)]
- [31. Membership cancellation in profile page]
---
- [32. Post views count]
- [33. Add Markdown editor in CREATE, MODIFY Post]
---
- 세션 타임아웃 지정
- 신규 댓글 알림 (short polling)


#### 사용 도구
|           |                |  버전  |
|-----------|----------------|--------|
| python    |                | 3.13.1 |
|           | Django         | 5.2.2  |
|           | django-environ | 0.12.0 |
| bootstrap |                | 5.3.x  |
| SimpleMDE |                | 1.11.2 |