# Closer look at the ruby scopes. 
	
In any programming language understanding scopes is very important. Scope defines visibility for variables, methods, classes, objects, constants. I’m going to discuss how ruby defines scopes for methods. Mos of the OO programming languages have three scopes for methods `public, private and protected`. Ruby also has `public, private and protected` scopes. I’m not going into the details of each scope. I’ll be discussing about how scope is defined for a method in ruby. 
	
```ruby
class Foo
    def public_method
        puts “I’m Public”
    end
    
    protected 
    def protected_method
	    puts “I’m protected”
    end

    private
    def private_method
	    puts “I’m private”
    end
end

class Bar < Foo
end

foo = Foo.new
foo.public_method #=> I'm Public
foo.protected_method #=> NoMethodError: protected method `protected_method' called for #<Foo>
foo.private_method #=>NoMethodError: private method `private_method' called for #<Foo>

bar = Bar.new
bar.public_method #=> I'm Public
bar.protected_method #=> NoMethodError: protected method `protected_method' called for #<Foo>
bar.private_method #=>NoMethodError: private method `private_method' called for #<Foo>
```

Let’s take a look at the above code, looks simple enough to understand right. Good, now let’s see the following code. 
```ruby
class Foo
    private
    def private_method
	puts “I’m private”
    end

    def self.private_method
	puts “I’m private”
    end
end

foo = Foo.new
foo.private_method #=> NoMethodError: private method `private_method' called for #<Foo>
Foo.private_method #=> I'm private
```

Some of you must have left wondering What changed? Why private_method is accessible for Foo class? Looking at the code, for those whom ruby is not first OO programming language above behavior of ruby must be weird. 

Most of the ruby developers comes from different OO programming language background, so they tend to ignore most of details about ruby language. We will see closely at ruby source code to understand how ruby defines scopes for methods. Let's start with what is `private` in ruby, is it a method or keyword or something else?

Let’s see. 

```c
/* vm_method.c */
void
Init_eval_method(void)
{
    ...
    rb_define_private_method(rb_cModule, "private", rb_mod_private, -1);
    ...
}
```

`private` is a method defined on Module class which is super class of class Class. When you write private inside body of class(Most of know body of class is a executable code in ruby) ruby calls ``rb_mod_private`` method. Which internally calls following two methods if you call private without any parameters then ``vm_cref_set_visibility`` is called to set the visibility of ``rb_vm_cref``(we'll discuss in detail). otherwise ``METHOD_ENTRY_VISI_SET`` this macro will be called to set the visibility of the method.

Let's follow the code for `vm_cref_set_visibility` method. 

```c
#vm_method.c
static void
vm_cref_set_visibility(rb_method_visibility_t method_visi, int module_func)
{
    rb_scope_visibility_t *scope_visi = (rb_scope_visibility_t *)&rb_vm_cref()->scope_visi;
    scope_visi->method_visi = method_visi;
    scope_visi->module_func = module_func;
}

#vm.c
rb_cref_t *
rb_vm_cref(void)
{
    rb_thread_t *th = GET_THREAD();
    rb_control_frame_t *cfp = rb_vm_get_ruby_level_next_cfp(th, th->cfp);

    if (cfp == NULL) {
	return NULL;
    }
    return rb_vm_get_cref(cfp->ep);
}
```

`vm_cref_set_visibility` function sets the scope visibility for `rb_vm_cref()`. 
`rb_vm_cref()` returns reference to the current object from the current control frame(we will discuss in detail about this in next blogpost).    

Now we will see how ruby defines a method and sets the scope of the method. 
```c
#vm.c
static void
vm_define_method(rb_thread_t *th, VALUE obj, ID id, VALUE iseqval, int is_singleton)
{
    VALUE klass;
    rb_method_visibility_t visi;
    rb_cref_t *cref = rb_vm_cref();

    if (!is_singleton) {
        klass = CREF_CLASS(cref);
        visi = rb_scope_visibility_get();
    }
    else { /* singleton */
        klass = rb_singleton_class(obj); /* class and frozen checked in this API */
        visi = METHOD_VISI_PUBLIC;
    }

    if (NIL_P(klass)) {
        rb_raise(rb_eTypeError, "no class/module to add method");
    }

    rb_add_method_iseq(klass, id, (const rb_iseq_t *)iseqval, cref, visi);

    if (!is_singleton && rb_scope_module_func_check()) {
        klass = rb_singleton_class(klass);
        rb_add_method_iseq(klass, id, (const rb_iseq_t *)iseqval, cref, METHOD_VISI_PUBLIC);
    }
}

#vm_method.c
static rb_method_visibility_t
rb_scope_visibility_get(void)
{
    rb_thread_t *th = GET_THREAD();
    rb_control_frame_t *cfp = rb_vm_get_ruby_level_next_cfp(th, th->cfp);

    if (!vm_env_cref_by_cref(cfp->ep)) {
        return METHOD_VISI_PUBLIC;
    }
    else {
        return CREF_SCOPE_VISI(rb_vm_cref())->method_visi;
    }
}
```

When you define method in ruby with `def` keyword, `vm_define_method` is called. If you look at the signature of this method. 
```c
static void
vm_define_method(rb_thread_t *th, VALUE obj, ID id, VALUE iseqval, int is_singleton);
```

`vm_define_method` accepts a flag called `is_singleton` which decides whether to add method on signleton class of a class(often called Eighenclass or virtual class) or add instance level methods. 

When you define method in following way. 
```ruby
class AnyClass
    def any_method
    end
end
```

Ruby calls `vm_define_method` with `is_singleton = false`, which means define instance level method. Now in that case visibility of the method is retrieved from `rb_scope_visibility_get` function. `rb_scope_visibility_get` function which retrieves visibility from `rb_vm_cref()` which is a reference to the current object in current control frame as we discussed earlier. 

But in case of of following code. 
```ruby
class AnotherClass
    def self.another_method
    end
end
```

Ruby calls `vm_define_method` with `is_singleton = true`, which means add method on signleton class of a class(often called Eighenclass or virtual class). Now in that case visibility of the method is set to `METHOD_VISI_PUBLIC`. 

Now I hope you understand it better how ruby defines visibility for methods. Let's understand if you want to define visibility for Eighneclass methods. 

```ruby
class AnyClass
    class << self
        def any_public_method
        end
        
        protected 
        def any_protected_method
        end
        
        private 
        def any_private_method
        end
    end
end
```

In above example of code, ruby will set scope in context of Eignenclass which will be used to set visibility on methods defined for Eignenclass.


