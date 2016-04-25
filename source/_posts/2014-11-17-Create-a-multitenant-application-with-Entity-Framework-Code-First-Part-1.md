---
title: Create a multitenant application with Entity Framework Code First - Part 1
date: 2014-11-17 23:15:00
tags: [entity framework, multi-tenant, interceptors, code first]
categories: [multitenant, application design, software as a service]
alias: multitenant/application%20design/software%20as%20a%20service/2014/11/17/create-a-multitenant-application-with-entity-framework-code-first---part-1.html
---


A highly increasing request we have to serve as developers, especially after Software as a Service revolution, is to provide software that is able to handle individual users in one application by separate each user's data. This feature is called [Multitenancy][mt] and there are several ways to achieve it. In this series of posts I will try to demonstrate how we can achieve the data isolation by using some advance features of Entity Framework. 
<!-- more --> 
***

#### The really simple model

The first part includes the description of the model used across the application. Lets assume we want to manage some Messages for our application and each logged in user should be able to access and modify only his personal messages. The Message is a classic [POCO][poco] class that doesn't have any Entity Framework specific code as you can see in the snippet below.
{% tabbed_codeblock Message POCO%}
    <!-- tab cs -->
public class Message {
    public long Id { get; set; }
    public string TenantId { get; set; }
    public virtual ApplicationUser User { get; set; }
    [Required]
    public string Description { get; set; }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

We follow mainly the convention over configuration approach Entity Framework provides and the only explicit configuration we have to apply in our Message class is to declare TenantId as foreign key for User property by adding the code that follows in the DbContext class we are going to use for data access.
{% tabbed_codeblock Entity configuration%}
    <!-- tab cs -->
protected override void OnModelCreating(DbModelBuilder modelBuilder) {
    base.OnModelCreating(modelBuilder);
    
    modelBuilder.Entity<Message>()
        .HasRequired(m => m.User)
        .WithMany()
        .HasForeignKey(m => m.TenantId)
        .WillCascadeOnDelete(true);
}
    <!-- endtab -->
{% endtabbed_codeblock %}

***

#### The "manual" approach 

I will first implement an MVC Controller that includes the logic of filtering based on TenantId and assign the correct TenantId when creating a new message. The code we have to write is more or less like the snippet below:
{% tabbed_codeblock Controller%}
    <!-- tab cs -->
[Authorize]
public class MessagesController : Controller {
    private ApplicationDbContext db = new ApplicationDbContext();

    private string UserId { get { return User.Identity.IsAuthenticated ? User.Identity.GetUserId() : string.Empty; }}

    public ActionResult Index() {
        // Tenant specific query logic
        var messages = db.Messages.Include(m => m.User).Where(m => m.TenantId == UserId);
        return View(messages.ToList());
    }

    public ActionResult Details(long? id) {
        // Tenant specific query logic
        Message message = db.Messages.SingleOrDefault(m => m.Id == id && m.TenantId == UserId);
        if (message == null) {
            return HttpNotFound();
        }
        return View(message);
    }

    public ActionResult Create([Bind(Include = "Id,Description")] Message message) {
        if (ModelState.IsValid) {
            // Assign tenant Id explicitly
            message.TenantId = UserId;
            db.Messages.Add(message);
            db.SaveChanges();
            return RedirectToAction("Index");
        }
        return View(message);
    }

    public ActionResult Edit(long? id) {
        // Tenant specific query logic
        Message message = db.Messages.SingleOrDefault(m => m.Id == id && m.TenantId == UserId);
        if (message == null) {
            return HttpNotFound();
        }
        return View(message);
    }

    public ActionResult Edit([Bind(Include = "Id,Description")] Message message) {
        if (ModelState.IsValid) {
            // Assign tenant Id explicitly
            message.TenantId = UserId;
            db.Entry(message).State = EntityState.Modified;
            db.SaveChanges();
            return RedirectToAction("Index");
        }
        return View(message);
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

I didn't include all required logic and also removed the delete functionality but the problem of always remember to filter the correct data is obvious. On top of this we must assign the TenantId property explicitly as the user should not be able to do something like this. This approach is of course a valid solution but requires a lot of repetition of the same logic throughout our application something that is always error prone. On top of this the entire application has knowledge of the TenantId concept. We could solve this problem by creating a base class, something like base Repository or similar approach, and hide this detail there. This would definitely improve the design but still when the base class is not enough we have to remember to apply the filtering.

***

#### Entity Framework Interceptors

[Entity Framework Interceptors][interceptors] is a very powerful feature added in version 6 of the framework and we can use them in order to hide the filtering and assign the correct TenantId in only one place. To be able to do that we must first write some infrastructure code.

***

#### TenantAwareAttribute and custom convention

The first step is to create an ordinary attribute class with which we can annotate our classes in order to get the Tenant functionality automatically. The code for the attribute class is shown below:

{% tabbed_codeblock Custom Attribute%}
    <!-- tab cs -->
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public class TenantAwareAttribute : Attribute {
    public const string TenantAnnotation = "TenantAnnotation";
    public const string TenantIdFilterParameterName = "TenantIdParameter";
    
    public string ColumnName { get; private set; }

    public TenantAwareAttribute(string columnName) {
        ColumnName = columnName;
    }

    public static string GetTenantColumnName(EdmType type) {
        MetadataProperty annotation = type.MetadataProperties.SingleOrDefault( 
            p => p.Name.EndsWith(string.Format("customannotation:{0}", TenantAnnotation)));

        return annotation == null ? null : (string)annotation.Value;
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

When using this attribute we have to declare which property of the class is the one responsible for TenantId. Moreover exposes a static helper method to get the the tenant column name based on the EdmType. 

The next step is to use this attribute by adding a custom [Entity Framework convention][conventions] to the DbContext class we are going to use for data access in our application. To do this we have to add the code snippet below on the DbContext class.

{% tabbed_codeblock Add attribute to configuration%}
    <!-- tab cs -->
protected override void OnModelCreating(DbModelBuilder modelBuilder) {
    base.OnModelCreating(modelBuilder);
    
    modelBuilder.Entity<Message>()
        .HasRequired(m => m.User)
        .WithMany()
        .HasForeignKey(m => m.TenantId)
        .WillCascadeOnDelete(true);
    
    var conv = new AttributeToTableAnnotationConvention<TenantAwareAttribute, string>(
                    TenantAwareAttribute.TenantAnnotation, (type, attributes) => attributes.Single().ColumnName);

    modelBuilder.Conventions.Add(conv);
}
    <!-- endtab -->
{% endtabbed_codeblock %}

***

#### Changes in Message class

After creating this infrastructure code the Message class can change and decorate it with the TenantAwareAttribute attribute as below:

{% tabbed_codeblock Changed Message POCO%}
    <!-- tab cs -->
[TenantAware("TenantId")]
public class Message {
    public long Id { get; set; }
    public string TenantId { get; private set; }
    public virtual ApplicationUser User { get; set; }
    [Required]
    public string Description { get; set; }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

As we can see here there is another important change in the class. The TenantId property now is private set which means we can't explicitly assign it from our code base. The last issue is quite important as we are not able to assign an invalid TenantId by mistake. Furthermore we don't have to take care of assign a TenantId at all.

***

This was the first part of a series of three posts. It includes the problem and the infrastructure code for what is coming on the next posts. For those that can't wait I have created a [project in Github][github] that contains the full code in a relatively change model. In the next post I will describe how we can use the code we already created to apply filtering based on TenantId always in a transparent way for our application.

### <a href="http://xabikos.com/multitenant/application%20design/software%20as%20a%20service/2014/11/18/create-a-multitenant-application-with-entity-framework-code-first---part-2.html">Part 2</a>

### <a href="http://xabikos.com/multitenant/application%20design/software%20as%20a%20service/2014/11/19/create-a-multitenant-application-with-entity-framework-code-first---part-3.html">Part 3</a>



[mt]: http://en.wikipedia.org/wiki/Multitenancy
[poco]: http://en.wikipedia.org/wiki/Plain_Old_CLR_Object
[interceptors]: http://msdn.microsoft.com/en-us/data/dn469464.aspx#BuildingBlocks
[conventions]: http://msdn.microsoft.com/en-us/data/jj819164.aspx
[github]: https://github.com/xabikos/EfMultitenant
