# TypeOrm-Relations-cheatsheet


> ## Many-to-one / one-to-many relations
Many-to-one / one-to-many is a relation where A contains multiple instances of B, but B contains only one instance of A. Let's take for example User and Photo entities. ```User``` can have multiple ``photos``, but each photo is owned by only one single user.


```import { Entity, PrimaryGeneratedColumn, Column, ManyToOne } from "typeorm"
import { User } from "./User"

@Entity()
export class Photo {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    url: string

    @ManyToOne(() => User, (user) => user.photos)
    user: User
}
```

```
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from "typeorm"
import { Photo } from "./Photo"

@Entity()
export class User {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    name: string

    @OneToMany(() => Photo, (photo) => photo.user)
    photos: Photo[]
}
```
> Here we added ```@OneToMany``` to the ```photos``` property and specified the target relation type to be ```Photo```. You can omit ```@JoinColumn``` in a ```@ManyToOne / @OneToMany``` relation. ```@OneToMany``` cannot exist without ```@ManyToOne```. If you want to use ```@OneToMany```, ```@ManyToOne``` is required. However, the inverse is not required: If you only care about the ```@ManyToOne``` relationship, you can define it without having ```@OneToMany``` on the related entity. Where you set ```@ManyToOne``` - its related entity will have "relation id" and foreign key.
This example will produce following tables:

```
+-------------+--------------+----------------------------+
|                         photo                           |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| url         | varchar(255) |                            |
| userId      | int(11)      | FOREIGN KEY                |
+-------------+--------------+----------------------------+
​
+-------------+--------------+----------------------------+
|                          user                           |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| name        | varchar(255) |                            |
+-------------+--------------+----------------------------+
```

## Example how to save such relation:
```
const photo1 = new Photo()
photo1.url = "me.jpg"
await dataSource.manager.save(photo1)
```

```
const photo2 = new Photo()
photo2.url = "me-and-bears.jpg"
await dataSource.manager.save(photo2)
```
```
const user = new User()
user.name = "John"
user.photos = [photo1, photo2]
await dataSource.manager.save(user)
```

> or alternatively you can do:
```
const user = new User()
user.name = "Leo"
await dataSource.manager.save(user)
```
```
const photo1 = new Photo()
photo1.url = "me.jpg"
photo1.user = user
await dataSource.manager.save(photo1)
```
```
const photo2 = new Photo()
photo2.url = "me-and-bears.jpg"
photo2.user = user
await dataSource.manager.save(photo2)
```
With  enabled you can save this relation with only one ```save``` call.
To load a user with photos inside you must specify the relation in ```FindOptions```:
```
const userRepository = dataSource.getRepository(User)
const users = await userRepository.find({
    relations: {
        photos: true,
    },
})

```
> // or from inverse side

```
const photoRepository = dataSource.getRepository(Photo)
const photos = await photoRepository.find({
    relations: {
        user: true,
    },
})
```

> Or using QueryBuilder you can join them:
```
const users = await dataSource
    .getRepository(User)
    .createQueryBuilder("user")
    .leftJoinAndSelect("user.photos", "photo")
    .getMany()
```
// or from inverse side
```
const photos = await dataSource
    .getRepository(Photo)
    .createQueryBuilder("photo")
    .leftJoinAndSelect("photo.user", "user")
    .getMany()
  ```  
With eager loading enabled on a relation, you don't have to specify relations in the find command as it will ALWAYS be loaded automatically. If you use ```QueryBuilder``` eager relations are disabled, you have to use ```leftJoinAndSelect``` to load the relation.








## Many-to-many relations

. What are many-to-many relations
. Saving many-to-many relations
. Deleting many-to-many relations
. Loading many-to-many relations
. Bi-directional relations
. Many-to-many relations with custom properties
## What are many-to-many relations

What are many-to-many relations

Many-to-many is a relation where A contains multiple instances of B, and B contains multiple instances of A. Let's take for example ```Question``` and ```Category``` entities. A question can have multiple categories, and each category can have multiple questions.

```
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm"

@Entity()
export class Category {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    name: string
}
```
```
import {
    Entity,
    PrimaryGeneratedColumn,
    Column,
    ManyToMany,
    JoinTable,
} from "typeorm"
import { Category } from "./Category"

@Entity()
export class Question {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    title: string

    @Column()
    text: string

    @ManyToMany(() => Category)
    @JoinTable()
    categories: Category[]
}

```

```@JoinTable()``` is required for ```@ManyToMany``` relations. You must put @JoinTable on one (owning) side of relation.
This example will produce following tables:
```
+-------------+--------------+----------------------------+
|                        category                         |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| name        | varchar(255) |                            |
+-------------+--------------+----------------------------+

+-------------+--------------+----------------------------+
|                        question                         |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| title       | varchar(255) |                            |
| text        | varchar(255) |                            |
+-------------+--------------+----------------------------+

+-------------+--------------+----------------------------+
|              question_categories_category               |
+-------------+--------------+----------------------------+
| questionId  | int(11)      | PRIMARY KEY FOREIGN KEY    |
| categoryId  | int(11)      | PRIMARY KEY FOREIGN KEY    |
+-------------+--------------+----------------------------+
```
## Saving many-to-many relations
With ```cascades``` enabled, you can save this relation with only one save call.

```
const category1 = new Category()
category1.name = "animals"
await dataSource.manager.save(category1)

const category2 = new Category()
category2.name = "zoo"
await dataSource.manager.save(category2)

const question = new Question()
question.title = "dogs"
question.text = "who let the dogs out?"
question.categories = [category1, category2]
await dataSource.manager.save(question)
```
## Deleting many-to-many relations
With ``cascades`` enabled, you can delete this relation with only one save call.
To delete a many-to-many relationship between two records, remove it from the corresponding field and save the record.

```
const question = dataSource.getRepository(Question)
question.categories = question.categories.filter((category) => {
    return category.id !== categoryToRemove.id
})
await dataSource.manager.save(question)
```

This will only remove the record in the join table. The question and categoryToRemove records will still exist.

## Soft Deleting a relationship with cascade
This example shows how the cascading soft delete behaves:
```
const category1 = new Category()
category1.name = "animals"

const category2 = new Category()
category2.name = "zoo"

const question = new Question()
question.categories = [category1, category2]
const newQuestion = await dataSource.manager.save(question)

await dataSource.manager.softRemove(newQuestion)
```
In this example we did not call save or softRemove for category1 and category2, but they will be automatically saved and soft-deleted when the cascade of relation options is set to true like this:

```
import {
    Entity,
    PrimaryGeneratedColumn,
    Column,
    ManyToMany,
    JoinTable,
} from "typeorm"
import { Category } from "./Category"

@Entity()
export class Question {
    @PrimaryGeneratedColumn()
    id: number

    @ManyToMany(() => Category, (category) => category.questions, {
        cascade: true,
    })
    @JoinTable()
    categories: Category[]
}

```
## Loading many-to-many relations
To load questions with categories inside you must specify the relation in FindOptions:
```
const questionRepository = dataSource.getRepository(Question)
const questions = await questionRepository.find({
    relations: {
        categories: true,
    },
})
```
> Or using QueryBuilder you can join them:
```
const questions = await dataSource
    .getRepository(Question)
    .createQueryBuilder("question")
    .leftJoinAndSelect("question.categories", "category")
    .getMany()
   ``` 
With eager loading enabled on a relation, you don't have to specify relations in the find command as it will ALWAYS be loaded automatically. If you use ```QueryBuilder``` eager relations are disabled, you have to use ```leftJoinAndSelect``` to load the relation.

## Bi-directional relations
Relations can be uni-directional and bi-directional. Uni-directional relations are relations with a relation decorator only on one side. Bi-directional relations are relations with decorators on both sides of a relation.
We just created a uni-directional relation. Let's make it bi-directional:

```
import { Entity, PrimaryGeneratedColumn, Column, ManyToMany } from "typeorm"
import { Question } from "./Question"

@Entity()
export class Category {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    name: string

    @ManyToMany(() => Question, (question) => question.categories)
    questions: Question[]
}

```
```
import {
    Entity,
    PrimaryGeneratedColumn,
    Column,
    ManyToMany,
    JoinTable,
} from "typeorm"
import { Category } from "./Category"

@Entity()
export class Question {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    title: string

    @Column()
    text: string

    @ManyToMany(() => Category, (category) => category.questions)
    @JoinTable()
    categories: Category[]
}
```

We just made our relation bi-directional. Note that the inverse relation does not have a ```@JoinTable``` . ```@JoinTable``` must be only on one side of the relation.
Bi-directional relations allow you to join relations from both sides using ```QueryBuilder``` :

```
const categoriesWithQuestions = await dataSource
    .getRepository(Category)
    .createQueryBuilder("category")
    .leftJoinAndSelect("category.questions", "question")
    .getMany()
    
```

## Many-to-many relations with custom properties

In case you need to have additional properties in your many-to-many relationship, you have to create a new entity yourself. For example, if you would like entities ```Post``` and ```Category``` to have a many-to-many relationship with an additional ```order``` column, then you need to create an entity ```PostToCategory``` with two ```ManyToOne``` relations pointing in both directions and with custom columns in it:

```
import { Entity, Column, ManyToOne, PrimaryGeneratedColumn } from "typeorm"
import { Post } from "./post"
import { Category } from "./category"

@Entity()
export class PostToCategory {
    @PrimaryGeneratedColumn()
    public postToCategoryId: number

    @Column()
    public postId: number

    @Column()
    public categoryId: number

    @Column()
    public order: number

    @ManyToOne(() => Post, (post) => post.postToCategories)
    public post: Post

    @ManyToOne(() => Category, (category) => category.postToCategories)
    public category: Category
}
```
Additionally you will have to add a relationship like the following to ```Post``` and ```Category``` :

```
// category.ts
...
@OneToMany(() => PostToCategory, postToCategory => postToCategory.category)
public postToCategories: PostToCategory[];

// post.ts
...
@OneToMany(() => PostToCategory, postToCategory => postToCategory.post)
public postToCategories: PostToCategory[];
```
