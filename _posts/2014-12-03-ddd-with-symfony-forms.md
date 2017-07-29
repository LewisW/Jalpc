---
layout: post
title: DDD - Using Symfony Forms without setters
---

## A quick primer on modeling entities in DDD

A strong theme in DDD, is that [setters are an anti-pattern](http://williamdurand.fr/2013/06/03/object-calisthenics/#9-no-getters/setters/properties). Instead, we should provide deliberate methods that perform the behaviour we want, and use them to change the state. I'll use an example from my work-in-progress talk on DDD:

```php
<?php
class Article {
    /**
     * @Assert\NotBlank()
     */
	private $title;
	
    /**
     * @Assert\NotBlank()
     */
	private $body;
	
    private $createdOn;
    
	/**
     * @Assert\Choice(choices = {"draft", "published"})
     */
    private $status;
    
	function setTitle($title) {
		$this->title = $title;
	}
	function setBody($body) {
		$this->body = $body;
	}
	function setCreatedOn($date) {
		$this->createdOn = $date;
	}
	function setStatus($status) {
		$this->status = $status;
	}
}

$entity = new Article();
$entity->setTitle("Setters are bad y'all");
$entity->setCreatedOn(new \DateTime());
$entity->setStatus('draft');
$entity->setBody('Every time you generate a setter a kitten dies');
?>
```

As you can see from the entity and how it's used, the business rules are performed outside of the entity. The entity is really just representing a data store, providing very little value over direct interaction with the database.

Instead, in DDD, we drop setters and force interaction with our entities through behavioural methods that mimic real-life actions:

``` php
<?php
  class Article {
    private $title;
    private $body;
    private $createdOn;
    private $status;
    
    public static function draft($title, $body) {
      $article = new self();
      $article->title = $title;
      $article->body = $body;

      $article->createdOn = new \DateTime();
      $article->status = 'draft';
      
      return $article;
    }
    
    public function publish() {
      $this->status = 'published';
    }
  }

  $entity = Article::draft(
    "Setters are bad y'all",
    'Every time you generate a setter a kitten dies'
  );
?>
```

As you can see from the above example, the entity provides much more value, and is much easier to use. We've encapsulated our business rules in to our entity, and locked direct state changes away from outside influence. This allows us to ensure our business rules are always applied to our entity, and stops them being spread out amongst our application.

## Enter Symfony forms

This works well, until we start going down the traditional Symfony form component usage. It's common practise to pass your entity directly to the form manager, which will then populate the entity with the data submitted from our forms and validate it.

Except, in order to do that, Symfony forms is going to need to set each property individually. This conflicts with our DDD mentality, since we're effectively letting the form dictate the state of our entity.

One solution, is to [use ```empty_data``` to lazily create our entity](http://williamdurand.fr/2013/12/16/enforcing-data-encapsulation-with-symfony-forms/). This works fine when we're creating new value objects, but it doesn't work on its own for editing entities.

## The problem extends to validation too
Lets go back to our Article example. Lets say we have a form that lets you update the status of our article. The traditional way to do this would be to bind a Symfony form to our Article entity, update the state using setters and then validate the entity. Ignoring the fact that we've removed all the setters from our entity, can anyone see an issue with the following form?

```php
<?php
class ArticleType extends AbstractType {

    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder->add(
            'status',
            'choice',
            [
                'choices' => [
                	'draft' => 'Draft',
                	'published' => 'Published'
                ]
            ]
        );
	}
    
    public function getName()
    {
        return 'article';
    }
}
?>
```

The answer is, when we try and validate our Article entity, it's going to validate all the attributes. Including the title and body, which aren't even being submitted in this form. The standard solution, is to create [validation groups](http://symfony.com/doc/current/book/validation.html#validation-groups). This just covers up the underlying problem (with a crap-load of annotations).

## Stop coupling forms to entities
In his blog post [Decoupling Symfony2 forms from Entities](http://verraes.net/2013/04/decoupling-symfony2-forms-from-entities/), Mathias Verraes shows us how to use Data transfer objects to decouple our entities from our forms. The idea is that rather than coupling our form directly to the entity, we use a very simple object to store *only* the data relating to the form. This gives us the separation that we need, meaning when our form is submitted we can take the data from that DTO and then apply it to our entity through its methods. It also means we can put the validation rules on to our DTO, avoiding the need for ugly validation groups.

## Mapping DTOs to our entities
Mathias only briefly goes in to the concept of using DTOs for forms, I'm hoping to expand on that and answer a few problems that come from this method.

Lets start by introducing a DTO for our previous example:

```php
<?php
class PublishArticle {
    /**
     * @Assert\Choice(choices = {"draft", "published"})
     */
    public $status;
    
    public $existingArticle;
}
?>
```

We can apply the validation directly to our DTO, meaning we can customise it for each context that it's in. We can also make the properties public, since we're deliberately using DTOs as just a data-store - we don't need encapsulation since there are no business rules in a DTO.

Since we're now passing our DTO to Symfony forms, we need the ability to convert it to and form our original object. We can create a command for this:

```php
<?php
class PublishArticleCommand {
    public function convertToDTO(Article $article)
    {
    	$publishArticle = new PublishArticle();
        $publishArticle->status = $article->getStatus();
        $publishArticle->existingArticle = $article;
	
    	return $publishArticle;
    }

    public function perform(PublishArticle $publishArticle)
    {
        if ($publishArticle->status == 'published') {
            $publishArticle->existingArticle->publish();
        }
        else {
            $publishArticle->existingArticle->unpublish();
        }
        
        return $publishArticle->existingArticle;
    }
}
?>
```

Don't forget to modify your controller to pass in the DTO instead of your entity, and to convert it back before processing it. For example:

```php
<?php
class PublishArticleController {
    public function publishAction(Request $request, Article $article)
    {
        $command = new PublishArticleCommand();
        $form = $this->createForm(new PublishArticleType(), $command->convertToDTO($article));
    	$form->handleRequest($request);
    	
        if ($form->isValid()) {
            // Perform the publish
            $command->perform($form->getData());
            
            // ... You can now persist your newly updated $article entity
        }
    }
}
?>
```

Using this technique, we're able to keep DDD concepts in a tool that was made for CRUD operations. It also actually forces us to think about what each form is doing, separate to our entity. And lastly, it lets us drop the need for complex validation groups. 

Please let me know your thoughts, I'm still refining the use of data transformers to manage DTOs in Symfony. I'm by no-means a DDD expert, so I welcome improvements.

**Update 09/09/2015**: Thanks to [@stof](https://twitter.com/stof70) for pointing out that my previous usage of Symfony's transformers weren't appropriate.
