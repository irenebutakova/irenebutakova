---
published: true
layout: post
title: Once near the Bridge, now closer to the problem it solves
---

Навеяно дискуссиями в группе...

*Саша*. В большинстве источников, которые я читал, паттерн Мост рассматривается на примере нескольких операционных систем и API-интерфейсов рисования для этих операционных систем. В других источниках паттерн рассматривается в контексте рисования геометрических фигур палитрой цветов.

*Паша*. На редкость полезные примеры.

*Саша*. Как думаете, не считая операционных систем и геометрических фигур, паттерн находит применение где-то еще?

*Маша*. Можем рассмотреть пример приложения по начислению и выплате заработной платы.

*Саша*. Где же там Мост увидим?

*Паша*. Спроектируем.

*Маша*. Начнем с того, что у нас есть тип Employee, которому надо подсчитать сумму к выплате.
А также есть несколько видов оплаты. Например, фиксированная и сдельная.

*Саша*. Можно сделать абстрактный тип PaymentType, у него будут деривативы - Salary и Commission. Employee будет содержать ссылку на абстрактный PaymentType и запрашивать расчет заработной платы. Какой же здесь Мост?

*Паша*. Моста нет, но Стратегия есть точно.

*Маша*. Действительно, Стратегия есть. Тогда давайте представим, что перед выплатой заработной платы нужно определить, а положена ли Employee выплата сегодня. Допустим, выплаты для Employee, работающих на сдельной основе, проводятся каждый понедельник. Также Employee могут получать бонусы Bonus по разной схеме. Скажем, СashBonus и NonCashBonus.

*Саша*. Тогда у нас появится еще несколько иерархий.

*Паша*. Несколько иерархий Стратегий.

*Саша*. Employee может спросить у планировщика выплат Scheduler, является ли сегодня днем выплаты,
и, если является, то запросить у PaymentType сумму начислений, а далее запросить у Bonus
сумму бонусов.

*Саша* Сколько иерархий Стратегий не добавляй, все равно выглядит как Стратегия.

*Маша*. А еще выглядит как нарушение принципа Tell-Don't-Ask.

*Саша*. И чем же наши Стратегии его нарушают?

*Петя*. Видимо, вместо того, чтобы запрашивать данные у объекта, мы должны сказать,
что сделать с объектом. That is all OOP.

*Саша*. Тогда, чтобы Employee не запрашивал данные, а раздавал указания, можно
создать тип Payroll, в который каждый тип Стратегии будет добавлять свои расчеты.

*Паша*. Тогда и пригодится Мост. Между Стратегиям. Заодно Шаблонный метод задействуем.

```c#
public abstract class PayrollTerminal
{
    private PayrollTerminal _next;

    public void Handle(Payroll payroll)
    {
        if (DoHandle(payroll))
            _next?.Handle(payroll);
    }

    public PayrollTerminal SetNext(PayrollTerminal terminal)
    {
        _next = terminal;
        return this;
    }

    protected abstract bool DoHandle(Payroll payroll);
}

public abstract class Scheduler : PayrollTerminal
{
}

public class Weekly : Scheduler
{
    protected override bool DoHandle(Payroll payroll)
    {
        <!--some paycheck handling-->
        return true;
    }
}

public class Biweekly : Scheduler
{
    protected override bool DoHandle(Payroll payroll)
    {
        <!--some paycheck handling-->
        return true;
    }
}

public abstract class PaymentType : PayrollTerminal
{
}

public class Salary : PaymentType
{
    protected override bool DoHandle(Payroll payroll)
    {
        <!--some paycheck handling-->
        return true;
    }
}

public class Commission : PaymentType
{
    protected override bool DoHandle(Payroll payroll)
    {
        <!--some paycheck handling-->
        return true;
    }
}

public abstract class Bonus : PayrollTerminal
{
}

public class CashBonus : Bonus
{
    protected override bool DoHandle(Payroll payroll)
    {
        <!--some paycheck handling-->
        return true;
    }
}

public class Employee
{
    private Scheduler _scheduler;
    private PaymentType _paymentType;
    private Bonus _bonus;

    public void SetScheduler(Scheduler scheduler) => _scheduler = scheduler;
    public void SetPaymentType(PaymentType type) => _paymentType = type;
    public void SetBonus(Bonus bonus) => _bonus = bonus;

    public PayrollTerminal GetBonus()
    {
        _bonus.SetNext(null);
        return _bonus;
    }

    public PayrollTerminal FromPaymentTypeTo(PayrollTerminal terminal)
    {
        _type.SetNext(terminal);
        return _type;
    }
  
    public PayrollTerminal FromSchedulerTo(PayrollTerminal terminal)
    {
        _scheduler.SetNext(terminal);
        return _type;
    }
}

PayrollTerminal start = employee.FromSchedulerTo(
                          employee.FromPaymentTypeTo(
                            ...
                             employee.FromDeductionTo(
                               employee.FromBonusTo(null))));
start.Handle(new Payroll(employee));
```


*Саша*. Получается, только Employee знает о своей конфигурации, а Мост работает на уровне абстракций. Ни одна из абстракций не знает обо всех остальных. Помещай любые абстракции, в любом порядке.

*Паша*. А кто будет ответственный за создание деривативов PayrollTerminal? Задействуем фабрику?

*Петя*. Что бы ты не задействовал, ясно одно. Кто создает Employee, тот создает его при помощи этих деривативов, а значит тесно с ними связан.

*Паша*. Так и сам Employee тесно связан с деривативами. Он знает, сколько есть разных терминалов, а также знает их типы. Tight-coupling.

*Даша*. А Пол говорил, что нужно, чтобы классы имели слабую зависимость от других классов, то есть не зависели от внешних изменений.

*Саша*. Значит, инвертируем зависимости.

*Маша*. Как вариант, вместо того, чтобы загружать Employee конкретными PayrollTerminal, можно дать
возможность терминалу самому определить, следует ли применить его к конкретному Employee.

*Саша*. Но мы же пока определили у Employee три терминала, а тут его нужно будет
прокатить через каждый терминал каждой иерархии?

*Витя*. Каждый из этих терминалов может очень быстро определить, нужно ли ему обработать запрос.
А если нет, он просто перенаправит запрос на следующий терминал. Low-coupling.

```c#
public abstract class PayrollTerminal
{
    private PayrollTerminal _next;

    public void Handle(Payroll payroll)
    {
      if (DoHandle(payroll))
          _next?.Handle(payroll);
    }

    public PayrollTerminal SetNext(PayrollTerminal terminal)
    {
        _next = terminal;
        return this;
    }

    protected abstract bool DoHandle(Payroll payroll);
}

public class Salaried : PayrollTerminal
{
    public Salaried(PayrollTerminal next) : base(next)
    {
    }

    protected override bool DoHandle(Payroll payroll)
    {
        if (payroll.Employee.Type == Employee.PaymentType.Salaried)
        {
            <!--some paycheck handling-->
        }
        return true;
    }
}

public class Employee
{
    public enum PaymentSheduler { Biweekly, Weekly}
    public enum PaymentType { Salaried, Commissioned }
    public enum PaymentDeductor { Taxes, Forfeit }
    public enum PaymentBonus { Cash, NonCash }

    public PaymentSheduler Scheduler { get; }
    public PaymentType Type { get; }
    public PaymentDeductor Deductor { get; }
    public PaymentBonus Bonus { get; }
}

public class PayrollDepartment
{
    private readonly IEnumerable<Employee> _employees;
    public PayrollTerminal PayrollTerminals { get; }

    public PayrollDepartment(IEnumerable<Employee> employees)
    {
        _employees = employees;
        PayrollTerminals = new Weekly(
                             new Monthly(
                               new Salaried(
                                 new Commissioned(
                                   new CashBonus(
                                     new ForfeitDeductor(
                                       new TaxesDeductor(
                                         new ...(null)
                                       )
                                     )
                                   )
                                 )
                               )
                             );
    }

    public void Calculate()
    {
        foreach (var employee in _employees)
        {
            PayrollTerminals.Handle(new Payroll(employee));
        }
    }
}
```

*Паша*. Да какой там low-coupling, ведь при добавлении нового дериватива, ты должен
перекомпилировать и задеплоить весь PayrollDepartment с Employee в придачу.

*Витя*. А при помощи Моста мы могли добавлять новые деривативы, не меняя ничего,
кроме фабрики, которая создает объект Employee.

*Саша*. Выходит, мы не избежим проблемы сцепления, какой бы подход ни выбрали.

*Даша*. Пол бы сказал, чтобы решить любую проблему в приложении, нужно добавить еще один
уровень абстракции.

*Петя*. Как бы нам в погоне за решением проблемы не придумать собственный
предметно-ориентированный язык.

*Маша*. DSL иногда действительно является лучшим способом решения сложной проблемы.

*Саша*. Как студент, я бы не взял на себя ответственность за выведение уровня абстракции
за пределы обычного программирования.

*Петя*. Боюсь, такая ответственность будет преследовать тебя до конца дней.


> Все герои вымышлены, полное или частичное сходство является совпадением.
