# 第12章 单元测试

**由Erik Kuefler撰写**
**由Tom Manshreck编辑**

上一章介绍了Google对测试进行分类的两个主要轴线：规模和范围。简而言之，规模指的是测试所消耗的资源和它被允许做的事情，范围指的是一个测试打算验证多少代码。虽然谷歌对测试规模有明确的定义，但范围往往有点模糊不清。我们用单元测试这个术语来指代范围相对较窄的测试，如单个类或方法的测试。单元测试通常是小规模的，但这并不总是如此。
在防止错误之后，测试最重要的目的是提高工程师的生产力。与范围更广的测试相比，单元测试有许多特性，使其成为优化生产力的绝佳方式。

- 根据谷歌对测试规模的定义，它们往往是小的。小型测试是快速和确定的，允许开发人员频繁地运行它们，作为他们工作流程的一部分，并获得即时反馈。
- 他们往往很容易与他们正在测试的代码同时编写，允许工程师将他们的测试集中在他们正在工作的代码上，而不需要建立和理解一个更大的系统。
- 他们促进高水平的测试覆盖率，因为他们是快速和容易编写的。高测试覆盖率使工程师有信心进行修改，他们不会破坏任何东西。
- 当他们失败的时候，他们往往很容易理解什么是错的，因为每个测试在概念上是简单的，并集中在系统的一个特定部分。
- 它们可以作为文档和例子，向工程师展示如何使用被测试的系统部分，以及该系统打算如何工作。

由于它们有很多优点，在Google写的大多数测试都是单元测试，作为一个经验法则，我们鼓励工程师把80%的单元测试和20%的范围更广的测试混合起来。这个建议，再加上编写单元测试的简易性和运行速度，意味着工程师要运行大量的单元测试--一个工程师在平均工作日内执行数千个单元测试（直接或间接）是很正常的。
由于单元测试在工程师的生活中占了很大一部分，所以谷歌非常重视测试的可维护性。可维护的测试是那些 "只是工作 "的测试：写完后，工程师不需要再考虑它们，直到它们失败，而且这些失败表明有明确原因的真正的错误。本章的主要内容是探讨可维护性的概念和实现它的技术。

## 可维护性的重要性

想象一下这个场景。Mary想给产品增加一个简单的新功能，并且能够快速实现，也许只需要几十行代码。但是当她去检查她的改动时，她从自动测试系统那里得到了满屏的错误。她花了一天的时间来逐一检查这些错误。在每个案例中，修改没有引入实际的错误，但打破了测试对代码内部结构的一些假设，需要对这些测试进行更新。通常情况下，她很难弄清楚这些测试首先要做的是什么，而她为修复这些测试而添加的黑客攻击使这些测试在未来更加难以理解。最终，本应是一个快速的工作，却要花费数小时甚至数天的时间来忙碌，扼杀了玛丽的工作效率，消磨了她的士气。
在这里，测试与它的预期效果正好相反，它耗尽了生产力，而不是提高生产力，同时也没有意义地提高被测代码的质量。这种情况太常见了，谷歌的工程师们每天都在与它斗争。没有什么灵丹妙药，但谷歌的许多工程师一直在努力开发一套模式和实践来缓解这些问题，我们鼓励公司的其他部门也这样做。
玛丽遇到的问题不是她的错，而且她也无法避免这些问题：糟糕的测试必须在签入之前被修复，以免它们给未来的工程师带来拖累。概括地说，她遇到的问题分为两类。首先，她所使用的测试是很脆弱的：它们在应对一个无害的、不相关的变化时，没有引入真正的bug而损坏。第二，测试不明确：在测试失败后，很难确定哪里出了问题，如何修复它，以及这些测试首先应该做什么。

## 防止脆性测试

正如刚才所定义的，脆性测试是指在面对不相关的程序代码变化时失败的测试，它并没有引入任何真正的bug。<sup>1</sup>在只有几个工程师的小代码库中，每次修改都要调整一些测试，这可能不是一个大问题。但是，如果一个团队经常写脆弱的测试，测试维护将不可避免地消耗团队越来越多的时间，因为他们不得不在不断增长的测试套件中梳理越来越多的故障。如果一套测试需要工程师为每一个变化进行手动调整，称其为 "自动化测试套件 "就有点牵强了。
1 请注意，这与片状测试略有不同，片状测试是在不改变生产代码的情况下非确定地失败。
脆弱的测试在任何规模的代码库中都会造成痛苦，但在谷歌的规模中，它们变得尤为严重。一个工程师在工作过程中，一天之内可能会轻易地运行数千个测试，而一个大规模的变化（见第22章）可能会引发数十万的测试。在这种规模下，即使是影响一小部分测试的假性故障也会浪费大量的工程时间。谷歌的团队在他们的测试套件的脆性方面有很大的不同，但我们已经确定了一些实践和模式，倾向于使测试对变化更加健壮。

### 努力实现不变的测试

在谈论避免脆性测试的模式之前，我们需要回答一个问题：在编写测试之后，我们应该多长时间需要修改一次？任何花在更新旧测试上的时间都是不能用在更有价值的工作上的时间。因此，理想的测试是不变的：在它写完之后，除非被测系统的需求发生变化，否则它永远不需要改变。
这在实践中是什么样子的呢？我们需要考虑工程师对生产代码所做的各种改变，以及我们应该如何期望测试对这些改变做出反应。从根本上说，有四种变化。
*纯粹的重构*
当工程师重构一个系统的内部结构而不修改其界面时，无论是为了性能、清晰度，还是其他原因，系统的测试都不应该改变。在这种情况下，测试的作用是确保重构没有改变系统的行为。在重构过程中需要改变的测试表明，要么变化影响了系统的行为，不是纯粹的重构，要么测试没有写在适当的抽象水平上。Google依靠大规模的变化（在第22章中描述）来做这样的重构，使得这种情况对我们特别重要。
*新功能*
当工程师向现有系统添加新的功能或行为时，系统的现有行为应该不受影响。工程师必须编写新的测试来覆盖新的行为，但他们不应该需要改变任何现有的测试。与重构一样，在添加新功能时，对现有测试的改变表明该功能的非预期后果或不适当的测试。
*Bug修复*
修复bug与添加新功能很相似：bug的存在表明初始测试套件中缺少一个案例，bug修复应该包括缺少的测试案例。同样，错误修复通常不需要对现有的测试进行更新。
*行为改变
改变系统的现有行为是一种情况，我们希望对系统的现有测试进行更新。请注意，这种变化往往比其他三种类型的测试要昂贵得多。一个系统的用户很可能依赖于其当前的行为，对该行为的改变需要与这些用户协调，以避免混乱或破坏。在这种情况下，改变一个测试表明我们正在破坏系统的一个明确的契约，而在前面的情况下的改变表明我们正在破坏一个非预期的契约。低级别的库往往会投入大量的精力来避免曾经的行为改变，以免破坏他们的用户。
这方面的启示是，在你写了一个测试之后，在你重构系统、修复错误或增加新功能时，你不应该再去碰这个测试。这种理解使得大规模地使用一个系统成为可能：扩展系统只需要写少量的与你所做的改变有关的新测试，而不是可能要触动每一个针对该系统写的测试。只有在系统行为中发生的破坏性变化才需要回去改变它的测试，在这种情况下，更新这些测试的成本相对于更新所有系统用户的成本来说，往往是很小的。

### 通过公共API进行测试

现在我们明白了我们的目标，让我们看看一些做法，以确保测试不需要改变，除非被测系统的需求改变。到目前为止，确保这一点的最重要的方法是编写测试，以与用户相同的方式调用被测系统；也就是说，针对其公共API而不是其实现细节进行调用。如果测试的工作方式与系统的用户相同，根据定义，破坏测试的变化也可能破坏用户。作为额外的奖励，这种测试可以作为用户的有用的例子和文档。
请看例12-1，它验证了一个事务并将其保存到数据库中。
*例12-1. 一个事务的API*.

```java
public void processTransaction(Transaction transaction) {
  if (isValid(transaction)) {
    saveToDatabase(transaction);
  }
}

private boolean isValid(Transaction t) {
  return t.getAmount() < t.getSender().getBalance();
}

private void saveToDatabase(Transaction t) {
  String s = t.getSender() + "," + t.getRecipient() + "," + t.getAmount();
  database.put(t.getId(), s);
}

public void setAccountBalance(String accountName, int balance) {
  // Write the balance to the database directly
}

public void getAccountBalance(String accountName) {
  // Read transactions from the database to determine the account balance
}
```

测试这段代码的一个诱人的方法是去掉 "私有 "可见性修饰符，直接测试实现逻辑，如例12-2所示。
*例12-2. 对交易API的实现的天真测试

```java
@Test
public void emptyAccountShouldNotBeValid() {
  assertThat(processor.isValid(newTransaction().setSender(EMPTY_ACCOUNT)))
    .isFalse();
}

@Test
public void shouldSaveSerializedData() {
  processor.saveToDatabase(newTransaction()
    .setId(123)
    .setSender("me")
    .setRecipient("you")
    .setAmount(100));
  assertThat(database.get(123)).isEqualTo("me,you,100");
}
```

这个测试与交易处理器的交互方式与真实用户的交互方式大不相同：它窥视系统的内部状态并调用系统API中没有公开的方法。因此，测试是脆弱的，几乎任何对被测系统的重构（如重命名其方法，将其分解为一个辅助类，或改变序列化格式）都会导致测试中断，即使这种变化对类的真正用户来说是看不见的。相反，同样的测试覆盖率可以通过只针对类的公共许可API进行测试来实现，如例12-3所示。<sup>2</sup>
2 这有时被称为 "先用前门 "原则。
*例12-3. 测试公共API*

```java
@Test
public void shouldTransferFunds() {
  processor.setAccountBalance("me", 150);
  processor.setAccountBalance("you", 20);

  processor.processTransaction(newTransaction()
    .setSender("me")
    .setRecipient("you")
    .setAmount(100));

  assertThat(processor.getAccountBalance("me")).isEqualTo(50);
  assertThat(processor.getAccountBalance("you")).isEqualTo(120);
}

@Test
public void shouldNotPerformInvalidTransactions() {
  processor.setAccountBalance("me", 50);
  processor.setAccountBalance("you", 20);

  processor.processTransaction(newTransaction()
    .setSender("me")
    .setRecipient("you")
    .setAmount(100));

  assertThat(processor.getAccountBalance("me")).isEqualTo(50);
  assertThat(processor.getAccountBalance("you")).isEqualTo(20);
}
```

根据定义，只使用公共API的测试是以其用户的方式来访问被测系统。这样的测试更现实，也更不脆，因为它们形成了明确的契约：如果这样的测试被破坏，就意味着系统的现有用户也将被破坏。只测试这些契约意味着你可以自由地对系统进行任何内部重构，而不必担心对测试进行繁琐的修改。
什么是 "公共API "并不总是很清楚，这个问题真正涉及到单元测试中的 "单元 "的核心。单元可以小到一个单独的功能，也可以大到由几个相关的包/模块组成的集合。当我们在这里说 "公共API "时，我们实际上是在谈论该单元暴露给拥有该代码的团队之外的第三方的API。这并不总是与某些编程语言提供的可见性概念一致；例如，Java中的类可能将自己定义为 "公共"，以便被同一单元中的其他包所访问，但并不打算供该单元之外的其他方使用。一些语言，如Python，没有内置的可见性概念（通常依赖于惯例，如在私有方法名称前加上下划线），而像Bazel这样的构建系统可以进一步限制谁被允许依赖由编程语言声明的公共API。
定义一个单元的适当范围，以及什么应该被认为是公共API，更多的是艺术而不是科学，但这里有一些经验法则。

- 如果一个方法或类的存在只是为了支持一两个其他的类（即，它是一个 "帮助类"），它可能不应该被认为是自己的单元，它的功能应该通过这些类而不是直接测试。
- 如果一个包或类被设计成任何人都可以访问，而不需要咨询它的所有者，那么它几乎肯定构成了一个应该被直接测试的单元，它的测试以与用户相同的方式访问该单元。
- 如果一个包或类只能由拥有它的人访问，但它被设计为提供在一系列情况下有用的一般功能（即，它是一个 "支持库"），它也应该被视为一个单元并直接测试。这通常会在测试中产生一些冗余，因为支持库的代码会被它自己的测试和用户的测试所覆盖。然而，这样的冗余是有价值的：如果没有它，如果库的一个用户（和它的测试）被删除，测试覆盖率就会出现缺口。

在谷歌，我们发现工程师有时需要被说服，通过公共API进行测试比针对实现细节进行测试要好。这种不情愿的态度是可以理解的，因为写测试的重点往往是你刚刚写的那段代码，而不是弄清楚这段代码是如何影响整个系统的。然而，我们发现鼓励这种做法是很有价值的，因为额外的前期努力在减少维护负担方面得到了许多倍的回报。对公共API进行测试并不能完全防止脆性，但这是你能做的最重要的事情，以确保你的测试只在系统发生意外变化时失败。

### 测试状态，而不是交互

测试通常依赖于实现细节的另一种方式，不是测试调用系统的哪些方法，而是如何验证这些调用的结果。一般来说，有两种方法来验证被测系统的行为是否符合预期。通过状态测试，你观察系统本身，看它在被调用后是什么样子。通过交互测试，你要检查系统是否对其合作者采取了预期的行动序列，以响应调用它。许多测试将执行状态和交互验证的组合。
交互测试往往比状态测试更脆，原因与测试一个私有方法比测试一个公共方法更脆的原因相同：交互测试检查一个系统是如何得到其结果的，而通常你应该只关心结果是什么。例12-4说明了一个使用测试替身（在第13章进一步解释）来验证系统如何与数据库交互的测试。
*例12-4. 脆性交互测试*

```java
@Test
public void shouldWriteToDatabase() {
  accounts.createUser("foobar");
  verify(database).put("foobar");
}
```

该测试验证了针对数据库API的特定调用，但有几种不同的方式可能出错。

- 如果被测系统中的一个错误导致记录在写入后不久就从数据库中删除，测试就会通过，尽管我们希望它失败。
- 如果被测系统被重构，调用一个稍微不同的API来写一个等效的记录，测试将失败，即使我们希望它通过。

如例12-5所示，直接对系统的状态进行测试就不那么脆了。
*例12-5. 针对状态的测试*

```java
@Test
public void shouldCreateUsers() {
  accounts.createUser("foobar");
  assertThat(accounts.getUser("foobar")).isNotNull();
}
```

这个测试更准确地表达了我们所关心的：被测系统与之交互后的状态。

交互测试出现问题的最常见原因是过度依赖mocking frameworks。这些框架可以很容易地创建测试替身，记录并验证针对它们的每个调用，并在测试中使用这些替身来代替真实对象。这种策略直接导致了脆弱的交互测试，因此我们倾向于使用真实对象而不是模拟对象，只要真实对象是快速和确定的。

### 编写清晰的测试

迟早，即使我们已经完全避免了脆性，我们的测试也会失败。失败是一件好事--测试失败为工程师提供了有用的信号，也是单元测试提供价值的主要方式之一。

测试失败的发生是两个原因之一：

* 被测试的系统有问题或不完整。这个结果正是测试的目的：提醒你注意错误，以便你能修复它们。
* 测试本身是有缺陷的。在这种情况下，被测试的系统没有任何问题，但测试的指定是不正确的。如果这是一个现有的测试，而不是你刚刚写的，这意味着测试是脆性的。上一节讨论了如何避免脆性测试，但很少有可能完全消除它们。

当测试失败时，工程师的首要工作是确定故障属于哪种情况，然后诊断出实际问题。工程师这样做的速度取决于测试的清晰程度。一个清晰的测试是指对于诊断故障的工程师来说，其存在的目的和失败的原因是非常明确的。如果测试失败的原因不明显，或者很难弄清楚最初写这些测试的原因，那么测试就无法达到清晰的效果。清晰的测试还能带来其他的好处，比如记录被测系统，更容易作为新测试的基础。

随着时间的推移，测试的清晰度变得非常重要。测试往往比编写测试的工程师的寿命更长，而且随着时间的推移，对系统的要求和理解会发生微妙的变化。一个失败的测试完全有可能是多年前由一个已经不在团队中的工程师写的，这样就没有办法弄清楚它的目的或如何修复它。这与不明确的生产代码形成鲜明对比，通常情况下，你可以通过查看调用代码的内容和删除代码后的故障来确定其目的。对于一个不明确的测试，你可能永远不会明白它的目的，因为删除测试除了（可能）在测试覆盖率中引入一个微妙的漏洞外，没有其他影响。

在最坏的情况下，这些晦涩难懂的测试最终会被删除，因为工程师不知道如何修复它们。删除这些测试不仅会在测试覆盖率上带来一个漏洞，而且还表明该测试在其存在的整个期间（可能是多年）一直提供零价值。

对于一个测试套件来说，随着时间的推移，它是有用的，重要的是该套件中的每个单独的测试是尽可能的清晰。本节探讨了测试的技术和思维方式，以达到清晰的目的。

### 使你的测试完整而简明

帮助测试实现清晰的两个高级属性是完整性和一致性。一个测试是完整的，当它的主体包含读者需要的所有信息，以了解它是如何得出结果的。当一个测试不包含其他分散注意力的或不相关的信息时，它就是简洁的。例12-6显示了一个既不完整也不简洁的测试。

*例12-6. 一个不完整且杂乱的测试*

```java
@Test public void shouldPerformAddition() { 
    Calculator calculator = new Calculator(new RoundingStrategy(), 
    "unused", ENABLE_COSINE_FEATURE, 0.01, calculusEngine, false);
	int result = calculator.calculate(newTestCalculation()); 					 	 assertThat(result).isEqualTo(5); 
    // Where did this number come from?
}
```

这个测试在构造函数中传递了很多不相关的信息，而测试的实际重要部分则隐藏在一个辅助方法中。通过澄清辅助方法的输入，测试可以变得更加完整，通过使用另一个辅助方法来隐藏构建计算器的无关细节，使测试更加简洁，如例 12-7 所示。

*例12-7. 一个完整、简洁的测试*

``` java
@Test
public void shouldPerformAddition() { 
	Calculator calculator = newCalculator();
	int result = calculator.calculate(newCalculation(2, Operation.PLUS, 3));
	assertThat(result).isEqualTo(5); 
}
```

我们稍后讨论的想法，特别是围绕代码共享，将与完整性和简洁性挂钩。特别是，如果违反DRY（Don't Repeat Yourself）原则，可以使测试更加清晰，这通常是值得的。记住：一个测试的主体应该包含理解它所需要的所有信息，而不包含任何不相关或不相干的信息。

### 测试行为，而非方法

许多工程师的第一直觉是试图将他们的测试结构与他们的代码结构相匹配，这样每个生产方法都有一个相应的测试方法。这种模式一开始很方便，但随着时间的推移，它导致了一些问题：随着被测试的方法越来越复杂，它的测试也越来越复杂，变得越来越难以推理。例如，考虑例12-8中的代码片段，它显示了一个交易的结果。

许多工程师的第一直觉是试图将他们的测试结构与他们的代码结构相匹配，这样每个生产方法都有一个相应的测试方法。这种模式一开始很方便，但随着时间的推移，它导致了一些问题：随着被测试的方法越来越复杂，它的测试也越来越复杂，变得越来越难以推理。例如，考虑例12-8中的代码片段，它显示了一个交易的结果。

*例12-8. 一个交易片段*

``` java
public void displayTransactionResults(User user, Transaction transaction) { 	
 	ui.showMessage("You bought a " + transaction.getItemName());
	if (user.getBalance() < LOW_BALANCE_THRESHOLD){ 
		ui.showMessage("Warning: your balance is low!");
	}
}
```

如例12-9所示，一个测试涵盖了该方法可能显示的两个信息，这并不罕见。

*例12-9. 一个方法驱动的测试*

```java
public void testDisplayTransactionResults() { 
	transactionProcessor.displayTransactionResults( 
	newUserWithBalance( 
		LOW_BALANCE_THRESHOLD.plus(dollars(2))), 
		new Transaction("Some Item", dollars(3)));
	assertThat(ui.getText()).contains("You bought a Some Item");
	assertThat(ui.getText()).contains("your balance is low");
}
```

对于这样的测试，很可能一开始测试只包括第一个方法。后来，当添加第二条信息时，一个工程师扩展了测试的内容（违反了我们前面讨论的不变的测试理念）。这种修改开创了一个不好的先例：随着被测方法变得越来越复杂，实现的功能越来越多，其单元测试也会变得越来越复杂，越来越难以操作。

问题是，围绕方法的测试框架会自然而然地鼓励不明确的测试，因为一个方法经常在内部做一些不同的事情，并且可能有几个棘手的边缘和角落案例。有一个更好的方法：与其为每个方法写一个测试，不如为每个行为写一个测试。行为是一个系统对其在特定状态下如何响应一系列输入的任何保证。行为通常可以用 "给定"、"当 "和 "然后 "来表达。"鉴于一个银行账户是空的，当试图从该账户取钱时，该交易被拒绝"。方法和行为之间的映射是多对多的：大多数不重要的方法实现了多个行为，而一些行为则依赖于多个方法的交互。前面的例子可以用行为驱动的测试来重写，如例12-10所介绍。

*例12-10. 一个行为驱动的测试*

```java
@Test 
public void displayTransactionResults_showsItemName() {
	transactionProcessor.displayTransactionResults( 
		New User(), new Transaction("Some Item"));
	assertThat(ui.getText()).contains("You bought a Some Item"); 
}
```

``` java
@Test 
public void displayTransactionResults_showsLowBalanceWarning() {
	transactionProcessor.displayTransactionResults(
		newUserWithBalance( 
		LOW_BALANCE_THRESHOLD.plus(dollars(2))),
        new Transaction("SomeItem",dollars(3)));
	assertThat(ui.getText()).contains("your balance is low");
}
```

分开单个测试所需的额外的模板是值得的，所产生的测试要比原来的测试更清晰。行为驱动的测试往往比面向方法的测试更清晰，有几个原因。首先，他们读起来更像自然语言，允许他们自然地理解，而不是需要费力地进行心理解析。其次，它们更清楚地表达了因果关系，因为每个测试的范围更有限。最后，每个测试都是简短的、描述性的，这使得人们更容易看到哪些功能已经被测试了，并鼓励工程师们增加新的精简的测试方法，而不是堆积在现有的方法上。

### 以强调行为为目的的构建测试

将测试看作是与行为而不是方法的耦合，会大大影响测试的结构。影响了他们的结构。每个行为都有三个部分：一个定义系统如何设置的 "给定 "部分，一个定义系统行动的 "何时 "部分，以及一个验证结果的 "当时 "部分。 当这种结构是明确的时候，测试是最清晰的。一些框架，如Cucumber和Spock等一些框架直接在给定/时间/时间中进行测试。其他语言可以使用空白和可选的注释来使结构清晰，如例12-11所示。

*例12-11. 一个结构良好的测试*

```java
@Test 
public void transferFundsShouldMoveMoneyBetweenAccounts() {
	// Given two accounts with initial balances of $150 and $20 
	Account account1 = newAccountWithBalance(usd(150));
	Account account2 = newAccountWithBalance(usd(20));
	// When transferring $100 from the first to the second account 
	bank.transferFunds(account1, account2, usd(100));
	// Then the new account balances should reflect the transfer
	assertThat(account1.getBalance()).isEqualTo(usd(50));
	assertThat(account2.getBalance()).isEqualTo(usd(120));
}
```

这种程度的描述在琐碎的测试中并不总是必要的，通常省略注释并依靠空白来使各部分清晰。然而，明确的注释可以使更复杂的测试更容易理解。这种方法使我们有可能在三个层次的粒度上阅读测试。

1. 读者可以先看一下测试方法的名称（在下面讨论），以获得对被测试行为的大致描述。
2. 如果这还不够，读者可以看一下给定/何时/何时的注释，以了解对行为的正式描述。
3. 最后，读者可以看一下实际的代码，准确地看到这种行为是如何表达的。

这种模式最常被违反的是在对被测系统的多次调用中穿插断言（即合并 "当 "和 "然后 "块）。以这种方式合并 "then "和 "when "块会使测试不那么清晰，因为它使人们难以区分正在执行的动作和预期结果。

当一个测试确实想验证一个多步骤过程中的每个步骤时，定义when/then块的交替序列是可以接受的。长的区块也可以用 "and "字来分割，使其更具描述性。例12-12显示了一个相对复杂的、行为驱动的测试是什么样子的。

```java
@Test 
public void shouldTimeOutConnections() { // Given two users 
    User user1 = newUser(); 
    User user2 = newUser();
	// And an empty connection pool with a 10-minute timeout 
    Pool pool = newPool(Duration.minutes(10));
	// When connecting both users to the pool 
    pool.connect(user1); 
    pool.connect(user2);
	// Then the pool should have two connections
    assertThat(pool.getConnections()).hasSize(2);
	// When waiting for 20 minutes 
    clock.advance(Duration.minutes(20));
	// Then the pool should have no connections
    assertThat(pool.getConnections()).isEmpty();
	// And each user should be disconnected
    assertThat(user1.isConnected()).isFalse();
    assertThat(user2.isConnected()).isFalse();
}
```

在编写这种测试时，要注意确保你不会无意中同时测试多个行为。每个测试应该只覆盖一个行为，绝大多数的单元测试只需要一个 "when "和一个 "then "块。

### 以被测试的行为命名测试

面向方法的测试通常以被测试的方法命名（例如，对 updateBalance 方法的测试通常称为 testUpdateBalance）。对于更加集中的行为驱动的测试，我们有更多的灵活性，并有机会在测试的名称中传达有用的信息。测试名称非常重要：它通常是失败报告中第一个或唯一一个可见的标记，所以当测试中断时，它是你发现问题的最好机会。它也是表达测试意图的最直接的方式。

一个测试的名字应该概括它所测试的行为。一个好的名字既能描述在系统上采取的行动，又能描述预期的结果。测试名称有时会包括额外的信息，如在对系统采取行动之前，系统的状态或其环境。一些语言和框架允许测试相互嵌套，并使用字符串命名，例如例12-13，其中使用了Jasmine，这样做比其他语言和框架更容易。

*例12-13. 一些嵌套命名模式的例子*

``` 
describe("multiplication", function() { 
	describe("with a positive number", function() { 
		var positiveNumber = 10;
		it("is positive with another positive number", function() {
			expect(positiveNumber * 10).toBeGreaterThan(0);
		});
		it("is negative with a negative number", function() {
			expect(positiveNumber * -10).toBeLessThan(0);
		}); });
		describe("with a negative number", function() { 
			var negativeNumber = 10;
			it("is negative with a positive number", function() {
				expect(negativeNumber * 10).toBeLessThan(0);
			});
			it("is positive with another negative number", function() {
				expect(negativeNumber * -10).toBeGreaterThan(0);
			}); 
		}); 
	});
```

其他语言要求我们在方法名称中编码所有这些信息，导致方法的命名模式如例12-14所示。

*例12-14. 一些示例方法的命名模式*

``` 
multiplyingTwoPositiveNumbersShouldReturnAPositiveNumber multiply_postiveAndNegative_returnsNegative 
divide_byZero_throwsException
```

像这样的名字比我们通常为生产代码中的方法所写的要啰嗦得多，但使用情况不同：我们从来不需要写代码来调用这些方法，而且它们的名字经常需要由人类在报告中阅读。因此，额外的说明是有必要的。

许多不同的命名策略是可以接受的，只要它们在一个测试类中使用一致。如果你被卡住了，一个好的技巧是尝试用 "should "这个词来开始测试名称。当与被测类的名称一起使用时，这种命名方案允许将测试名称作为一个句子来阅读。例如，一个名为shouldNotAllowWithdrawalsWhenBalanceIsEmpty的BankAccount类的测试可以被理解为 "BankAccount不应该允许在余额为空时提款"。通过阅读套件中所有测试方法的名称，你应该对被测系统实现的行为有一个很好的感觉。这样的名称也有助于确保测试集中在一个单一的行为上：如果你需要在测试名称中使用 "and "字，很有可能你实际上是在测试多个行为，应该编写多个测试。

### 不要把逻辑放在测试中

清晰的测试在检查时是微不足道的，也就是说，只需看一眼，就能看出一个测试在做正确的事情。这在测试代码中是可能的，因为每个测试只需要处理一组特定的输入，而生产代码必须被泛化以处理任何输入。对于生产代码，我们能够编写测试，确保复杂的逻辑是正确的。但测试代码没有这种奢侈--如果你觉得你需要写一个测试来验证你的测试，那就说明出了问题！这是不可能的。

复杂性最常以逻辑的形式引入。逻辑是通过编程语言的指令性部分来定义的，如运算符、循环和条件等。当一段代码包含逻辑时，你需要做一些心理计算来确定其结果，而不是仅仅从屏幕上读出来。不需要太多的逻辑就可以使一个测试变得更难推理。例如，例12-15中的测试在你看来是否正确？

*例12-15. 掩盖错误的逻辑*

```java
@Test 
public void shouldNavigateToAlbumsPage() { 
    String baseUrl = "http://photos.google.com/"; 
    Navigator nav = new Navigator(baseUrl); 
    nav.goToAlbumPage(); 
    assertThat(nav.getCurrentUrl()).isEqualTo(baseUrl + "/albums");
}
```

这里没有什么逻辑：实际上只是一个字符串连接。但是，如果我们通过删除这一点逻辑来简化测试，一个错误就会立刻变得清晰，如例12-16所示。

```java
@Test 
public void shouldNavigateToPhotosPage() { 
	Navigator nav = new Navigator("http://photos.google.com/");
    nav.goToPhotosPage(); 
    assertThat(nav.getCurrentUrl()))
        .isEqualTo("http://photos.google.com//albums"); // Oops!
}
```

当整个字符串被写出来的时候，我们可以马上看到，我们在URL中期望有两个斜线，而不是只有一个。如果生产代码犯了类似的错误，这个测试将无法检测到一个错误。为了使测试更有描述性和意义，重复基本的URL仅仅花费很小的代价（见本章后面关于DAMP与DRY测试的讨论）。

如果人们不善于发现来自字符串连接的错误，那么我们更不善于发现来自更复杂的编程结构的错误，如循环和条件。这个道理十分明确：在测试代码中，坚持使用直线代码而不是清晰的逻辑，并考虑容忍一些冗余，当它使测试更具有描述性和意义时。我们将在本章后面讨论关于冗余和代码共享的想法。

### 编写清晰的故障信息

清晰度的最后一个方面与测试的编写方式无关，而是与工程师在测试失败时看到的内容有关。在一个理想的情况下，工程师可以通过阅读日志或报告中的失败信息来诊断一个问题，而不需要看测试本身。一个好的故障信息包含与测试名称相同的信息：它应该清楚地表达预期结果、实际结果和任意相关的参数。