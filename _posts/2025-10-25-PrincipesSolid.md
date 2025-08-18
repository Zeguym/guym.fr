---
title: Principes SOLID en .NET
tags: solid architecture 
---

Les principes **S.O.L.I.D** sont un ensemble de bonnes pratiques en programmation orientée objet qui permettent de concevoir des systèmes logiciels robustes, flexibles et maintenables. Ces principes sont particulièrement utiles en .NET pour structurer des applications modulaires et testables.

<!--more-->

## S.O.L.I.D : Les 5 principes

### 1. 'S' - Single Responsibility Principle (SRP)

**Une classe ne doit avoir qu'une seule responsabilité.**

- Une classe doit avoir **une seule raison de changer**.
- Cela signifie qu'une classe doit se concentrer sur une seule tâche ou fonctionnalité.

**Exemple en .NET :**

```csharp
public class Invoice
{
    public void CalculateTotal() { /* Calculer le total */ }
}

public class InvoicePrinter
{
    public void PrintInvoice(Invoice invoice) { /* Imprimer la facture */ }
}
```

- Ici, la classe `Invoice` est responsable des calculs, tandis que `InvoicePrinter` est responsable de l'impression. Cela respecte le SRP.

### 2. 'O' - Open/Closed Principle (OCP)

**Une classe doit être ouverte à l'extension, mais fermée à la modification.**

- Vous devez pouvoir ajouter de nouvelles fonctionnalités sans modifier le code existant.
- Cela peut être réalisé via l'héritage ou les interfaces.

**Exemple en .NET :**

```csharp
public interface IDiscount
{
    decimal ApplyDiscount(decimal amount);
}

public class NoDiscount : IDiscount
{
    public decimal ApplyDiscount(decimal amount) => amount;
}

public class PercentageDiscount : IDiscount
{
    public decimal ApplyDiscount(decimal amount) => amount * 0.9m;
}

public class Invoice
{
    private readonly IDiscount _discount;

    public Invoice(IDiscount discount)
    {
        _discount = discount;
    }

    public decimal GetTotal(decimal amount)
    {
        return _discount.ApplyDiscount(amount);
    }
}
```

- Ici, vous pouvez ajouter de nouveaux types de remises (`IDiscount`) sans modifier la classe `Invoice`.

## 3. 'L' - Liskov Substitution Principle (LSP)

**Une classe dérivée doit pouvoir être utilisée à la place de sa classe de base sans altérer le comportement du programme.**

- Les sous-classes doivent respecter le contrat de la classe de base.

**Exemple en .NET :**

```csharp
public class Bird
{
    public virtual void Fly() { Console.WriteLine("Flying"); }
}

public class Sparrow : Bird { }

public class Penguin : Bird
{
    public override void Fly()
    {
        throw new NotSupportedException("Penguins can't fly");
    }
}
```

- Ici, `Penguin` viole le principe LSP, car il ne peut pas être utilisé comme un `Bird` dans un contexte où `Fly` est attendu. Une meilleure solution serait de repenser la hiérarchie.

### 4. 'I' - Interface Segregation Principle (ISP)

**Une interface ne doit pas forcer une classe à implémenter des méthodes inutilisées.**

- Les interfaces doivent être spécifiques à un rôle, et non générales.

**Exemple en .NET :**

```csharp
public interface IPrinter
{
    void Print();
    void Scan();
    void Fax();
}

public class BasicPrinter : IPrinter
{
    public void Print() { /* Imprimer */ }
    public void Scan() { throw new NotImplementedException(); }
    public void Fax() { throw new NotImplementedException(); }
}
```

- Ici, `BasicPrinter` viole l'ISP, car il est forcé d'implémenter des méthodes inutiles. Une meilleure solution :

```csharp
public interface IPrinter
{
    void Print();
}

public interface IScanner
{
    void Scan();
}

public class BasicPrinter : IPrinter
{
    public void Print() { /* Imprimer */ }
}
```

### 5. 'D' - Dependency Inversion Principle (DIP)

**Les modules de haut niveau ne doivent pas dépendre des modules de bas niveau. Les deux doivent dépendre d'abstractions.**

- Les classes doivent dépendre d'interfaces ou d'abstractions, et non de classes concrètes.

**Exemple en .NET :**

```csharp
public class EmailService
{
    public void SendEmail(string message) { /* Envoyer un email */ }
}

public class Notification
{
    private readonly EmailService _emailService;

    public Notification()
    {
        _emailService = new EmailService();
    }

    public void Notify(string message)
    {
        _emailService.SendEmail(message);
    }
}
```

- Ici, `Notification` dépend directement de `EmailService`, ce qui viole le DIP. Une meilleure solution :

```csharp
public interface IMessageService
{
    void SendMessage(string message);
}

public class EmailService : IMessageService
{
    public void SendMessage(string message) { /* Envoyer un email */ }
}

public class Notification
{
    private readonly IMessageService _messageService;

    public Notification(IMessageService messageService)
    {
        _messageService = messageService;
    }

    public void Notify(string message)
    {
        _messageService.SendMessage(message);
    }
}
```

- Maintenant, `Notification` dépend de l'abstraction `IMessageService`, et non de l'implémentation concrète. Dans une application ASP.NET Core, c'est ensuite le conteneur d'injection de dépendances qui fournit l'implémentation concrète d'`IMessageService`.

## Résumé des principes SOLID

| **Principe** | **Description**                                                               |
| ------------ | ----------------------------------------------------------------------------- |
| **S**        | Une classe doit avoir une seule responsabilité.                               |
| **O**        | Une classe doit être ouverte à l'extension, mais fermée à la modification.    |
| **L**        | Les sous-classes doivent pouvoir remplacer leurs classes de base.             |
| **I**        | Les interfaces doivent être spécifiques à un rôle.                            |
| **D**        | Les modules doivent dépendre d'abstractions, pas d'implémentations concrètes. |

## Avantages des principes SOLID

- **Réduction du couplage** : Les classes sont moins dépendantes les unes des autres.
- **Facilité de test** : Les classes peuvent être testées indépendamment.
- **Extensibilité** : Le code est plus facile à étendre sans modifier les classes existantes.
- **Maintenance** : Le code est plus lisible et plus facile à maintenir.

## Mise en pratique dans un projet .NET

Pour appliquer progressivement SOLID dans un projet existant, vous pouvez par exemple :

- Identifier les classes « god object » (trop de responsabilités) et les découper en plusieurs classes plus ciblées.
- Introduire des interfaces là où une classe de haut niveau dépend directement d'une implémentation concrète, puis brancher ces abstractions dans le conteneur d'injection de dépendances.
- Profiter d'un refactoring ou d'une nouvelle fonctionnalité pour vérifier, principe par principe, si le code que vous modifiez respecte bien SRP, OCP, LSP, ISP et DIP.

En appliquant ces principes dans vos projets .NET, vous pouvez concevoir des systèmes logiciels robustes, évolutifs et maintenables.
