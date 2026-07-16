---
author: ["Joseph Ward"]
title: "Test Archaeology: Following Natural/Adabas Behaviour from the Mainframe to AWS"
description: "Using AI and Cucumber to trace Natural/Adabas execution paths during DVLA's mainframe-to-AWS migration."
draft: false
date: 2026-07-16
tags: ["Ruby", "Testing", "Cucumber", "AI", "AWS", "Natural", "Adabas", "Migration"]
categories: ["Testing", "Ruby"]
ShowToc: true
TocOpen: true
---

# Test Archaeology: Following Natural/Adabas Behaviour from the Mainframe to AWS

D90 is a long‑running Natural/Adabas application that supports core driver and vehicle licensing services. We’re moving it from its existing IBM mainframe environment to AWS, which raises an obvious question: will it still work the same way?

The application launched in the 1990s and has been refined ever since. Moving it to AWS requires adapting to a different platform: encoding, sort order, I/O behaviour, and other assumptions built into the code over decades. We are adding Cucumber scenarios using Software AG’s browser-based Natural interface with Capybara and Playwright to ensure key features work as expected.

AI has been a big help in deciding which scenarios to cover with specifications by example written in Ruby.

I call this work "test archaeology" because you rarely start with clear rules. Instead, you dig through historic behaviour, source code, execution paths, and outputs to understand what’s going on.

## Starting with behaviour

In every scenario, the same setup and action produce the same result, whether showing a value, giving a message, making a decision, or changing a state.

The browser interface offers a common view and reference point. Even as the underlying platform changes, the app's appearance and function must remain familiar to users.

Character order, fixed-width fields, padding, formatting, and unexpected value combinations can all reveal platform assumptions. Look at the code underneath to confirm what’s really happening.

## Following the execution path

Searching through source code might surface something interesting, but its value to the migration depends on whether a real journey reaches it.

Natural has several ways to structure execution. You can put logic right inside a program, bring it in with an INCLUDE copy at compile time, or run it in a subprogram that you call with CALLNAT. One subprogram can even call another. The name you pass to CALLNAT might come from a variable, so just looking for program names won't always show the full path. Software AG explains these behaviours in the [`INCLUDE`](https://documentation.softwareag.com/natural/nat912unx/sm/include.htm) and [`CALLNAT`](https://documentation.softwareag.com/natural/nat912unx/sm/callnat.htm) statements.

Sometimes, the same business rule appears inline in another program. In a long-running, reactive system like ours, multiple routes can produce similar behaviour, with context spread across source code, data definitions, and the people who have known the application the longest.

So we start at the browser journey and trace inwards. A broad pass identifies entry programs, callers, callees, included copycode, related Adabas fields and similar inline logic. A deep pass follows the parameters and conditions controlling one path until we can connect an input to an observable result.

Names and familiar patterns provide search terms. We verify their meaning through the code and running application.

## Prompts for investigation

What we've talked about so far needs a different kind of prompt than just "write tests for this program." At this point, we're still figuring out how relevant the code is and what the test should cover.

A test-archaeology prompt should leave a clear path for others to follow. It needs to separate facts from guesses, keep track of open questions, and suggest what to look for next. Flagging what we don't know is as useful as confirming what we do.

I find it useful to think of discoverability and researchability as two different things. Discoverability is about finding the code and data behind a process. Researchability is about showing how we came to a conclusion and what we still don't know. The test comes from the parts we can be sure about.

A prompt might look like this:

```text
Investigate the Natural/Adabas behaviour behind this browser
journey: [journey and observed outcome].

Treat source code as evidence. Do not infer business intent from
program names, naming conventions or familiar design patterns.

Trace broadly:

- identify entry programs, callers and callees;
- locate literal and variable CALLNAT targets;
- expand INCLUDE copycode and nested copycode;
- find PERFORM, FETCH and other execution dependencies;
- locate similar conditions implemented inline elsewhere; and
- record the Adabas fields and other inputs controlling each path.

Trace each relevant path deeply:

- record every condition that gates execution;
- follow values and transformations across program boundaries;
- identify encoding, collation, padding and fixed-width assumptions;
- record state changes and visible results; and
- retain a source reference for every claim.

Classify each finding as confirmed, inferred or unresolved.

For every reachable branch, describe the potential business outcome
if its behaviour changes after replatforming. Record any input that
is difficult to arrange or result that is difficult to observe.

Return an execution map, an evidence log, unresolved searches and
candidate Cucumber scenarios linked to confirmed paths.
```

The output is research material for the next pass. We review the sources, follow unresolved links and discuss the business meaning with somebody who understands that part of the service.

## A small Natural example

The following fictional code illustrates why finding a platform-sensitive line and its path are part of the same task.

```natural
** Enquiry result ordering
IF #RECORDS-FOUND > 1
  CALLNAT 'DEQSRT01' #RESULT-ARRAY(*) #SORT-COUNT
END-IF
```

The subprogram might rely on character comparison to sort results:

```natural
DEFINE DATA PARAMETER
1 #RESULT-ARRAY (A30/1:50)
1 #SORT-COUNT (N3)
LOCAL
1 #I (N3)
1 #J (N3)
1 #TEMP (A30)
END-DEFINE

FOR #I = 1 TO #SORT-COUNT - 1
  FOR #J = #I + 1 TO #SORT-COUNT
    IF #RESULT-ARRAY(#J) < #RESULT-ARRAY(#I)
      MOVE #RESULT-ARRAY(#I) TO #TEMP
      MOVE #RESULT-ARRAY(#J) TO #RESULT-ARRAY(#I)
      MOVE #TEMP TO #RESULT-ARRAY(#J)
    END-IF
  END-FOR
END-FOR
```

The `<` comparison on alphanumeric fields depends on the platform's character ordering. EBCDIC sorts letters before numbers, ASCII sorts numbers before letters. So a field with mixed content (like a reference code) comes out in a different order on each platform, and the user sees results rearranged.

This is one example of a wider problem: a full EBCDIC codebase can contain platform assumptions in digit checks, punctuation handling, sorting, print formatting, padding, number representation and input/output behaviour, any of which could change after migration. The way we deal with each assumption is the same: find where it appears, understand the path that reaches it, and check the observable behaviour carefully.

But knowing the character ordering differs isn't enough on its own. We still need to find which browser action calls `DEQSRT01`, what makes `#RECORDS-FOUND > 1` true, how the result array gets populated, and what the user actually sees.

You might also find the same sort logic written inline in another program. That's a different path with its own inputs, conditions, and context.

## Turning the path into a Cucumber scenario

Once the path and outcome are clear, we write the scenario. Named cases keep the Gherkin table readable and the report output clean.

```gherkin
Feature: Casework enquiry result ordering after replatforming

  Scenario Outline: Results appear in the expected order for "<description>"
    Given casework records exist for driver "<driver>"
    When I search for the driver
    Then the results should appear in this order:
      | reference   |
      | <first>     |
      | <second>    |
      | <third>     |

    Examples:
      | description          | driver  | first    | second   | third    |
      | mixed alpha-numeric  | JONES01 | 1AB-2024 | 9AB-2024 | AAB-2024 |
      | leading zeros        | JONES02 | 001-REF  | 010-REF  | 100-REF  |
      | special characters   | JONES03 | O'B-2024 | OBR-2024 | OBS-2024 |
```

In practice, the step definitions might look something like this, adapted to our framework's session handling and page objects:

```ruby
COLLATION_REFERENCES = {
  "JONES01" => %w[1AB-2024 9AB-2024 AAB-2024], # digits vs letters
  "JONES02" => %w[001-REF 010-REF 100-REF],    # leading zeros
  "JONES03" => %w[O'B-2024 OBR-2024 OBS-2024]  # apostrophe position
}.freeze

Given(/^casework records exist for driver '(.*)'$/) do |driver|
  D90Helpers.create_driver_with_casework(driver, references: COLLATION_REFERENCES.fetch(driver))
  @driver_key = driver
end

When(/^I search for the driver$/) do
  Navigator.to('casework_enquiry')
  fill_in 'Driver', with: @driver_key
  click_button 'Search'
end

Then(/^the results should appear in this order:$/) do |table|
  expected = table.hashes.map { |row| row["reference"] }
  actual = all("[data-test='result-reference']").map(&:text)
  expect(actual).to eq(expected)
end
```

The route, labels, selectors, and values are just placeholders. The actual ones come from the path we followed. Playwright takes care of the interaction and waiting, while the scenario explains the comparison we're focusing on.

## Choosing useful test questions

A simple branch count is not a test plan. We're looking for cases where the migration could change a result someone depends on.

For every path we can reach, we ask how a platform difference might change things. Could it pick a different record? Could it affect a decision or a change in state? Could padding or conversion alter a value shown later on? These questions help us pick data with a reason behind it.

Negative testing needs just as much care. "Invalid" can mean a lot of things, like missing data, wrong format, out of range, not matching other data, or just not fitting the process at the right time. AI can help sort these kinds of issues, but someone has to decide which of those differences really matter and add useful information.

When we trace, we note down testability limits. Can we make the data we need? Can we check the result through the interface? Does the path rely on shared state or timing? Do two branches end up with the same visible result? These questions might lead us to change the setup, make a more focused check, or make other assertions.

The goal is to have a set of scenarios where each one clearly shows what it's comparing and why that comparison is important for the replatform.

## What the tests leave behind

The suite's main job right now is to make sure the migration goes smoothly. Each scenario shows us how the system behaves on both platforms.

It also leaves behind an executable description of the application. The scenario records its setup, action and outcome. The research behind it — execution maps, evidence logs, unresolved questions — records the programs, data and decisions that connect them. Those artefacts sit alongside the test code so that the next person to touch a journey can see not just *what* we tested but *why*, and what we ruled out.

Over time, this accumulates into something more valuable than either the tests or the documentation alone. A future change starts with a known journey, a traced path through the source, and a record of which platform assumptions were confirmed. That's a different starting point from another blank search box.

AI helps us cover more ground and keep our findings organised. The archaeology never stops, but each pass leaves a better map than we started with.