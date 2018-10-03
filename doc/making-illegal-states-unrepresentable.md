*Представляю вашему вниманию перевод статьи Scott Wlaschin ["Designing with types: Making illegal states unrepresentable"](https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/).*

В этой статье мы рассмотрим ключевое преимущество F# - возможность "сделать некорректные состояния невыразимыми" при помощи системы типов (фраза заимствована у [Yaron Minsky](https://blog.janestreet.com/effective-ml-revisited/)).

Рассмотрим тип `Contact`. В результате [проведённого рефакторинга](https://fsharpforfunandprofit.com/posts/designing-with-types-single-case-dus/) он сильно упростился:

```fsharp
type Contact = 
    {
    Name: Name;
    EmailContactInfo: EmailContactInfo;
    PostalContactInfo: PostalContactInfo;
    }
```

Теперь предположим, что существует простое бизнес-правило: "Контакт должен содержать адрес электронной почты или почтовый адрес". Соответствует ли наш тип этому правилу?

Нет. Из правила следут, что контакт может содержать адрес электронной почты, но не иметь почтового адреса, или наоборот. Однако в текущем виде тип требует, чтобы были заполнены оба поля.

Кажется, ответ очевиден - сделать адреса необязательными, например, так:

```fsharp
type Contact = 
    {
    Name: PersonalName;
    EmailContactInfo: EmailContactInfo option;
    PostalContactInfo: PostalContactInfo option;
    }
```

Но теперь наш тип допускает слишком многое. В этой реализации можно создать контакт вообще без адреса, хотя правило требует, чтобы хотя бы один адрес был указан.

Как же решить эту задачу?
<cut />

# Как сделать некорректные состояния невыразимыми

Обдумав правило бизнес-логики можно прийти к выводу, что возможны три случая:

* указан только адрес электронной почты;
* указан только почтовый адрес;
* указан и адрес электронной почты, и почтовый адрес.

В такой формулировке решение становится очевидным - сделать тип-сумму с конструктором для каждого возможного случая.

```fsharp
type ContactInfo = 
    | EmailOnly of EmailContactInfo
    | PostOnly of PostalContactInfo
    | EmailAndPost of EmailContactInfo * PostalContactInfo

type Contact = 
    {
    Name: Name;
    ContactInfo: ContactInfo;
    }
```

Эта реализация полностью соответствует требованиям. Все три случая выражены явно, при этом четвёртый случай (без какого-либо адреса) не допускается.

Обратите внимание на случай "адрес электронной почты и почтовый адрес". Пока что я просто использовал кортеж. В данном случае этого достаточно.

## Создание `ContactInfo`

Теперь давайте посмотрим, как использовать эту реализацию на примере. Для начала созданим новый контакт:

```fsharp
let contactFromEmail name emailStr = 
    let emailOpt = EmailAddress.create emailStr
    // обработка случаев с корректным и некорректным адресом электронной почты
    match emailOpt with
    | Some email -> 
        let emailContactInfo = 
            {EmailAddress=email; IsEmailVerified=false}
        let contactInfo = EmailOnly emailContactInfo 
        Some {Name=name; ContactInfo=contactInfo}
    | None -> None

let name = {FirstName = "A"; MiddleInitial=None; LastName="Smith"}
let contactOpt = contactFromEmail name "abc@example.com"
```

В этом примере мы создаём простую вспомогательную функцию `contactFromEmail`, чтобы создать новый контакт, передав имя и адрес электронной почты. Однако адрес может быть некорректным, и функция должна обрабатывать оба этих случая. Функция не может создать контакт с некоректным адресом, поэтому она возвращает значени типа `Contact option`, а не Contact.

## Изменение `ContactInfo`

Если надо добавить почтовый адрес к существующему `ContactInfo`, то придётся обработать три возможных случая:

* если у контакта был только адрес электронной почты, то теперь у него указаны оба адреса, поэтому надо вернуть контакт с конструктором `EmailAndPost`;
* если у контакта был только почтовый адрес, надо вернуть контакт с конструктором `PostOnly`, заменив почтовый адрес на новый;
* если у контакта были оба адрес, надо вернуть контакт с конструктором `EmailAndPost`, заменив почтовый адрес на новый.

Вспомогательная функция для обновления почтового адреса выглядит следующим образом. Обратите внимание на явную обработку для каждого случая.

```fsharp
let updatePostalAddress contact newPostalAddress = 
    let {Name=name; ContactInfo=contactInfo} = contact
    let newContactInfo =
        match contactInfo with
        | EmailOnly email ->
            EmailAndPost (email,newPostalAddress) 
        | PostOnly _ -> // существующий почтовый адрес игнорируется
            PostOnly newPostalAddress 
        | EmailAndPost (email,_) -> // существующий почтовый адрес игнорируется
            EmailAndPost (email,newPostalAddress) 
    // создать новый контакт
    {Name=name; ContactInfo=newContactInfo}
```

А вот так выглядит использование этого кода:

```fsharp
let contact = contactOpt.Value   // обратите внимание на предупреждение касательно option.Value ниже
let newPostalAddress = 
    let state = StateCode.create "CA"
    let zip = ZipCode.create "97210"
    {   
        Address = 
            {
            Address1= "123 Main";
            Address2="";
            City="Beverly Hills";
            State=state.Value; // обратите внимание на предупреждение касательно option.Value ниже
            Zip=zip.Value;     // обратите внимание на предупреждение касательно option.Value ниже
            }; 
        IsAddressValid=false
    }
let newContact = updatePostalAddress contact newPostalAddress
```

*ПРЕДУПРЕЖДЕНИЕ: В этом примере я использовал `option.Value`, чтобы получить содержимое option. Это допустимо, когда вы экспериментируете в интерактивной консоли, но это ужасное решение для рабочего кода! Надо всегда использовать [сопоставление с образцом](https://docs.microsoft.com/ru-ru/dotnet/fsharp/language-reference/pattern-matching) и обрабатывать оба конструктора `option`.*

# Зачем заморачиваться этими сложными типами?

К этому времени вы могли решить, что мы всё слишком усложнили. Отвечу тремя тезисами.

Во-первых, бизнес-логика сложна сама по себе. Простого способа этого избежать нет. Если ваш код проще бизнес-логики, вы не обрабатываете все случаи, как надо.

Во-вторых, если логика выражена типами, то она самодокументируется. Можно посмотреть на конструкторы типа-суммы ниже и сразу понять бизнес-правило. Вам не придётся тратить время на анализ какого-либо другого кода.

```fsharp
type ContactInfo = 
    | EmailOnly of EmailContactInfo
    | PostOnly of PostalContactInfo
    | EmailAndPost of EmailContactInfo * PostalContactInfo
```

Наконец, если логика выражена типом, то любые изменения правил бизнес-логики сломают код, не учитывающий эти изменения, а это, как правило, хорошо.

Последний пункт раскрывается в [следующей статье](https://fsharpforfunandprofit.com/posts/designing-with-types-discovering-the-domain/). Пытаясь выразить правила бизнес-логики через типы, вы можете прийти к углублённому пониманию предметной области.
