# Python Live Project - Code Retrospective

## Table of Contents
* <a href="#intro">Introduction</a>
* <a href="#travelogue">Create a Travelogue App</a>
* <a href="#accounts">Improve User Account Management</a>
* <a href="#budget">Add Functionality to the Budget App</a>
* <a href="#summary">Overall Summary</a>

### <span id="intro">Introduction</span>
During a two week sprint, I worked with a team of peers on a web application that facilitates a variety of microservices. The primary languages I used during this project were Python, HTML, CSS and the Django framework. Along with those languages I also utilized Git for version control, Azure DevOps for project tracking, and Slack to communicate with my team. The user stories I completed involved both front and back end work, which provided a rounded development experience. My primary focus was back end, so I used Bootstrap to quickly create user friendly interfaces for my projects that could be easily updated later if need be.

### <span id="travelogue">Building the Travelogue App</span>
The largest feature I implemented was building a travel blog. The user could add title, location, blog content, and tags to their post. The author and date were automatically added by user authentication and current DateTime. Functionality also included editing posts, searching by tags for related content, and the ability for users to add comments.

Here are the models I created. Categories (or tags) are a many to many field because each post can have numerous categories. Comments are a separate model with posts as a forgeign key. 

``` python
class Category(models.Model):
    name = models.CharField(max_length = 20)

    def __str__(self):
        return self.name

class Post(models.Model):
    id = models.AutoField(primary_key = True)
    title = models.CharField(max_length = 100)
    author = models.ForeignKey(User, on_delete = models.CASCADE) #Use Django's auth model to assign the logged in user as author
    location = models.CharField(max_length = 50)
    body = models.TextField()
    created_on = models.DateTimeField(auto_now_add = True)
    last_modified = models.DateTimeField(auto_now = True)
    categories = models.ManyToManyField('Category', related_name = 'posts') # Categories are used as tags

    def __str__(self):
        return self.title

class Comment(models.Model):
    author = models.CharField(max_length = 50)
    body = models.TextField()
    created_on = models.DateTimeField(auto_now_add = True)
    post = models.ForeignKey('Post', on_delete = models.CASCADE)

    def __str__(self):
        return self.author
```

And the views to render content and manage forms:
```python
# Index view of the travelogue. Returns a preview of all blogs posts, sorted by descending date
def travelogue_index(request):
    posts = Post.objects.all().order_by('-created_on')
    context = {
        "posts": posts,
    }
    return render(request, 'TravelogueApp/travelogue.html', context)

# Search for all posts that have a chosen category(AKA "tag"), and order them by descending date
def travelogue_search(request, category):
    posts = Post.objects.filter(
        categories__name__contains = category
        ).order_by('-created_on')
    context = {
        "category": category,
        "posts": posts
    }
    return render(request, 'TravelogueApp/travelogue_search.html', context)

# Render the full details of a post (given its primary key) with the comments section
def travelogue_detail(request, pk):
    post = Post.objects.get(pk=pk)

    form = CommentForm()
    if request.method == 'POST':
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = Comment(
                author = form.cleaned_data["author"], # For now, the user can enter an author name
                body = form.cleaned_data["body"],
                post = post # Assign the comment to its corresponding blog post
            )
            comment.save()
            return redirect('travelogue_detail', pk=post.pk)

    comments = Comment.objects.filter(post=post).order_by('-created_on') # Show all comments related to this post, sorted by date
    context = {
        "post": post,
        "comments": comments,
        "form": form,
    }
    return render(request, 'TravelogueApp/travelogue_detail.html', context)

# Render the Post ModelForm to create a new travelogue post
def travelogue_create(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False) # Prevent the form from immediately saving so the author and categories can be updated properly
            post.author = request.user # Assign the author field to the account logged in
            post.save() # The post object (with form data) must be saved before the many to many field ('categories') is saved
            form.save_m2m() # This saves the selected categories from the form to the new post
            return redirect('travelogue_detail', pk=post.pk) # Once the form/post is saved, redirect to the post's details page
    else:        
        form = PostForm() # If the request is GET, render an empty form for the user to fill out
    return render(request, 'TravelogueApp/travelogue_create.html', {'form': form})

# Render the Post ModelForm to edit a travelogue post
def travelogue_edit(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if request.method == 'POST':
        form = PostForm(request.POST, instance=post)
        if form.is_valid():
            post = form.save(commit=False)
            post.save() # The post object (with form data) must be saved before the many to many field ('categories') is saved
            form.save_m2m() # This saves the selected categories from the form to the new post
            return redirect('travelogue_detail', pk=post.pk) # Once the form/post is saved, redirect to the post's details page
    else:        
        form = PostForm(instance=post) # If the request is GET, render the existing blog post data
    return render(request, 'TravelogueApp/travelogue_edit.html', {'form': form})

# Render the form to add new categories for use on blog posts
def travelogue_categories(request):    
    categories = Category.objects.all().order_by('name')
    form = CategoryForm()
    if request.method == 'POST':
        form = CategoryForm(request.POST)
        if form.is_valid():
            newCategory = Category(
                name = form.cleaned_data['name']
            )            
            newCategory.save()
            return redirect('travelogue_categories')
    else:
        context = {
            'form': form,
            'categories': categories,
        }
    return render(request, 'TravelogueApp/travelogue_categories.html', context)	
```

The home page of the Travelogue App lists each blog post the user has created, with a snip of the content. Here is part of the HTML:
```html
<div class="col-med-8 mt-3">
    {% for post in posts %}
        <div class="card travelogue-spacing">
            <div class="card-body">
                <p class="card-title"><a class="blog-header" href="{% url 'travelogue_detail' post.id %}">{{ post.title }}</a></p>
                <p class="card-subtitle text-muted">
                    Author: {{ post.author }} |&nbsp;
                    {{ post.created_on.date }} |&nbsp;
                    Location: {{ post.location }} |&nbsp;
                    Tags:&nbsp;
                    {% for category in post.categories.all %}
                        <u><a class="blog-tag" href="{% url 'travelogue_search' category.name %}">{{ category.name }}</a></u>&nbsp;
                    {% endfor %}
                </p>
                <hr>
                <p>{{ post.body | slice:":400" }}...</p><a class="btn-sm btn-info" href="{% url 'travelogue_detail' post.id %}">Continue Reading</a>
            </div>
        </div>
    {% endfor %}
</div>
```

This is the Travelogue details page, so you can view a post in its entirety, search by tags, and leave/view comments. 
![Travelogue detail page](https://github.com/jrs-scott/Python-Back-End-Code-Retrospective/blob/master/blog-detail-page.JPG)

### <span id="accounts">Improve User Account Management</span>
When I started work on the app, there was the ability for a user to log in, but that's it. I improved the account management by adding the ability for a user to create an account, log out, and reset their password via email if they forgot it. 

I went with a clean and straightforward design approach for the log in, create, and forgot password templates. 
![Travel Scrape log in page](https://github.com/jrs-scott/Python-Back-End-Code-Retrospective/blob/master/login-page.JPG)

The views for managing accounts: 
```python
def dashboard(request):
    return render(request, 'AccountsApp/dashboard.html')

# Create and verify a new user, then log them in and redirect to the dashboard
# Include required email to create user form so they have the ability to reset their passwords
def signup(request):
    if request.method == 'POST':
        form = SignUpForm(request.POST)
        if form.is_valid():
            form.save()
            username = form.cleaned_data.get('username')
            email = form.cleaned_data.get('email')
            raw_password = form.cleaned_data.get('password1')
            user = authenticate(username=username, email=email, password=raw_password)
            login(request, user)
            return redirect('/') 
    else:
        form = SignUpForm()
    return render(request, 'registration/signup.html', {'form': form})    

# Log in form. Redirects verified user to the dashboard
def login_view(request):
    if request.method == 'POST':
        username = request.POST['username']
        password = request.POST['password']
        user = authenticate(request, username=username, password=password)
        if user is not None:
            login(request, user)
            return redirect('/') 
    else:
        return render(request, 'AccountsApp/dashboard.html')

# Log out the user and render custom logout.html page
def logout_view(request):
    logout(request)
    return render(request, 'registration/logout.html')

# View with form for user to enter their email to have a password link sent to them
def password_reset(request):
    return render(request, 'registration/password_reset_form.html')

# View to confirm the password reset link was sent
def password_reset_done(request):
    return render(request,'registration/password_reset_done.html')
```

The rest of my task was creating pages for account creation, password reset, and logout. I didn't include those here for sake of brevity.


### <span id="budget">Extend Functionality of the Budget App</span>
A BudgetApp had been created, but required improvements. My user story was to update the app's UI, add the ability to delete a budget, edit its allotted amount, edit and delete expenses, and add sub navigation within the app. To improve user experience, I started with UI changes. Keeping a similar theme to the Travelogue app, I kept the navigation the same spot on every page so the user never had to search for links. I implemented Bootstrap modals for the edit forms because it was simple and effective. 

Here is the index view of the app, which lists the user's current budgets. Depending on if the budget was at, below, or above $0, the text would change color and give a different description.
![Budget App index view](https://github.com/jrs-scott/Python-Back-End-Code-Retrospective/blob/master/budget-list.JPG)

The most time consuming portion was working on the budget details page. There was a lot going on, including three forms that needed to be identified and processes appropriately. To do so, I added hidden inputs on each form that gave each one a unique identifier. Within the views, I used logic flow statements to dictate which form to handle. To pass the object's values into the form inputs, I used JavaScript.

The detail view code:
```python
# Render the detail page of a specific budget, given its unique id
# This view includes creating/editing/deleting expenses, editing the budget allotted amount, and deleting the entire budget
def project_detail(request, id):
    project = get_object_or_404(Project, id=id)
    expense_list = project.expenses.all()

    # Categories are specific to each budget/project and are referenced as foreign keys
    if request.method == 'GET':
        category_list = Category.objects.filter(project=project)
        context = {
            'project': project, 
            'expense_list': expense_list, 
            'category_list': category_list
        }
        return render(request, 'BudgetApp/project-detail.html', context)

    # Each form on this page has a hidden input field with a form-type that helps identify it for the post requests
    elif request.method == 'POST':
        # Add an expense to the budget
        if request.POST['form-type'] == 'add-expense-form':
            form = ExpenseForm(request.POST)
            if form.is_valid():
                title = form.cleaned_data['title']
                amount = form.cleaned_data['amount']
                category_name = form.cleaned_data['category']

                # Categories are models that have project as a foreign key, so each category is an object
                category = get_object_or_404(Category, project=project, name=category_name)

                Expense.objects.create(
                    project=project,
                    title=title,
                    amount=amount,
                    category=category
                ).save()

        # Edit the total allotted budget amount. Currently this is done via a Bootstrap modal
        elif request.POST['form-type'] == 'edit-budget-form':
            form = EditBudgetForm(request.POST)
            if form.is_valid():
                project.budget = form.cleaned_data['budget']
                project.save()

        # Update an expense from the budget expense table. Currently this is done via a Bootstrap modal
        elif request.POST['form-type'] == 'edit-expense-form':
            expense = get_object_or_404(Expense, id=request.POST['expenseId'])
            form = ExpenseForm(request.POST)
        
            if form.is_valid():
                expense.title = form.cleaned_data['title']
                expense.amount = form.cleaned_data['amount']
                expense.category = get_object_or_404(Category, project=project, name=form.cleaned_data['category'])
                expense.save()

    # Delete an expense from the budget
    elif request.method == 'DELETE':
        id = json.loads(request.body)['id']
        expense = get_object_or_404(Expense, id=id)
        expense.delete()
        return HttpResponse('')
    
    return HttpResponseRedirect(reverse('detail', args=(project.id,)))
```

When all was said and done, the details page looked like this. Each edit button brought up a modal that would update the database information when posted. 
![Budget App details view](https://github.com/jrs-scott/Python-Back-End-Code-Retrospective/blob/master/budget-detail.JPG)


## <span id="summary">Summary</span>
Working on this project gave me a lot of practical experience including:
* Completing user stories on deadlines
* Communicating with team members and the project manager to stay on the same page
* Further exposure to the Django framework and working within a large scale project
* Using Git for version control
* Exposure to Azure DevOps for project management
* Learning from other developers and working together to resolve technical issues
* Researching and project planning for a new application
