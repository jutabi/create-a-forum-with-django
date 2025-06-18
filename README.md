# [개인 프로젝트 - Django로 게시판 만들기]

<details>
    <summary>커밋 내역</summary>
    1. 프로젝트 생성
    2. urls.py, views.py 체험
    3. Model 생성
    4. superuser 생성 (Django Admin)
    5. READ Posts, Comments
    6. CREATE Comment
    7. CREATE Post
    8. UPDATE Post / use template filter
    9. DELETE Post
    10. UPDATE Comment
    11. DELETE Comment
    12. DELETE Post, Comment with using html-dataset
    13. Isolate script file (Django static)
    14. Apply Bootstrap
    15. Template inheritance
    16. Apply Django Form (use is_valid)
    17. Append navbar
    18. Paging Index page (post list)
    19. Isolate form_errors.html
    20. Sign in / Sign out
    21. Sign up
    22. Password find, change
    23. Add Author in Post, Comment model's properties
    24. Redirect to sign_in page when unknown user doing? CREATE post, comment
    25. Show author in Post, Comment
    26. UPDATE, DELETE only for author
    27. Separate 'views.py' by function
    28. Add recommend (vote) function
    29. Use HTML anchor when comment CREATE, MODIFY, VOTE
    30. Search Post (basic 'title' search)
    31. Search Post (input 'search_target')
    32. Password reset (basic setting using Django PasswordResetView)
    33. Secure secret keys using django-environ
    34. Password change (redirect log_in page when users not logged in access django password change)
    35. Member's profile page (add href in post, comment's created user)
    36. Profile setting in navbar dropdown menu
    37. Post views count
    38. Add Markdown editor (SimpleMDE)
    39. Recommend (vote) Ajax
    40. Comment create Ajax
        1. basic setting
        2. print errors without Django From.errors context
        3. send page where the comment is written
    41. Comments Paging with Ajax
    42. Comment UPDATE Ajax
    43. Comment DELETE Ajax
</details>

#### 프로젝트 목차
- [01. Initial setting](/docs/01-Initial-setting.md)
- [02. Create Model](/docs/02-model.md)
- [03. Django Administrator](/docs/03-Django-administrator.md)
- [04. READ Posts, Post](/docs/04-READ-Post.md)
- [05. CREATE Post](/docs/05-CREATE-Post.md)
- [06. UPDATE Post](/docs/06-UPDATE-Post.md)
- [07. DELETE Post](/docs/07-DELETE-Post.md)
- [08. Django static, bootstrap](/docs/08-Template-inheritance.md)
- [09. Template inheritance](/docs/08-Template-inheritance.md)
10. Navigation bar
11. Post list paging
12. SignIn / SignOut
13. SignUp / Membership cancellation
14. Add attribute(author) to Post
15. Show author in post_detail
16. Only the author can UPDATE, DELETE
17. Vote function with Async (Ajax)
18. urls.py structure (Flat, Nested, shallow routing)
19. CREATE Comment SSR + number of comments
20. CREATE Comment with Async (Fetch)
21. READ Comments with JsonResponse
22. UPDATE Comment with Async (Fetch)
23. DELETE Comment with Async (Fetch)
24. Separate 'views.py' by function
25. Search post (basic 'title' search)
26. Search post (input 'search_target')
27. Password reset (with using Django PasswordResetView)
28. Password reset (secure secret keys with using django-environ)
29. Password change
30. Member's profile (show created posts, comments)
31. Navbar dropdown menu
32. Post views count
33. Add Markdown editor in CREATE, MODIFY Post
34. Comments paging


#### 사용 도구
|           |                |  버전  |
|-----------|----------------|--------|
| python    |                | 3.13.1 |
|           | Django         | 5.2.2  |
|           | django-environ | 0.12.0 |
| bootstrap |                | 5.3.x  |
| SimpleMDE |                | 1.11.2 |