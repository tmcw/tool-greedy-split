<script lang="ts">
  type Steps = {
    description: string;
    after: Record<string, number>;
  };
  type Sums = {
    name: string;
    sum: number;
    items: Item[];
  };
  type Item = {
    name: string;
    value: number;
    trailer: string;
    error: string | undefined;
  };

  let freetext = $state(`Bill: $81
Bob: $74
Ted: $57`);

  const currencyFormat = new Intl.NumberFormat("en-US", {
    style: "currency",
    currencyDisplay: "symbol",
    currency: "USD",
  });

  function assert(test: unknown, msg: string) {
    if (!test) {
      throw new Error(msg);
    }
  }

  let items = $derived.by<Item[]>(() => {
    return freetext
      .split("\n")
      .map((l) => l.trim())
      .filter((l) => l)
      .map((l) => {
        let errors: string[] = [];
        const [name, _after] = l.split(":");
        const [_value, trailer] = _after?.trim().split(/\s/g, 2);
        const _strippedValue = _value.replace(/[\$\,]/g, "");
        const value = Number.parseFloat(_strippedValue);
        if (Number.isNaN(value)) {
          errors.push(
            `Couldn't parse this line's amount: ${l} (after: ${_after}), (value: ${_value} - ${_strippedValue})`,
          );
        }
        return {
          name,
          value,
          trailer,
          error: errors.join(", "),
        };
      });
  });

  let totals = $derived.by<Sums[]>(() => {
    return Array.from(
      Map.groupBy(items, (item) => item.name).entries(),
      ([name, items]) => {
        return {
          name,
          items,
          sum: items.reduce((sum, i) => i.value + sum, 0),
        };
      },
    );
  });

  let total = $derived.by<number>(() => {
    return totals.reduce((sum, t) => t.sum + sum, 0);
  });

  let steps = $derived.by<Steps[]>(() => {
    let currentTotals = Object.fromEntries(totals.map((t) => [t.name, t.sum]));

    let targetPerPerson = total / totals.length;

    let maxSteps = 50;
    let count = 0;
    let epsilon = 0.1;
    let steps: Steps[] = [];

    while (true) {
      let sorted = Object.entries(currentTotals).sort((a, b) => a[1] - b[1]);

      // The person who owes the least
      let least = sorted.at(0)!;
      // Who owes the most
      let most = sorted.at(-1)!;

      // How much the person who owest the most needs to be paid back
      let payment = Math.min(
        most[1] - targetPerPerson,
        targetPerPerson - least[1],
      );

      assert(
        payment >= 0,
        `The person who owes the most should never owe negative: owes ${payment}`,
      );

      currentTotals[least[0]] += payment;
      currentTotals[most[0]] -= payment;

      steps.push({
        description: `${least[0]} pays ${most[0]} ${currencyFormat.format(payment)}`,
        after: currentTotals,
      });

      let done = Object.entries(currentTotals).every(
        (t) => Math.abs(t[1] - targetPerPerson) < epsilon,
      );

      if (done) break;

      if (count++ > maxSteps) break;
    }

    return steps;
  });
</script>

<h1>Greedy split</h1>

<p>
  Simplified implementation of the required math to settle up debts. This
  accepts a syntax in which each line is a person's name, a :, and a dollar
  amount. Like <code>Bob: $10</code>
</p>

<textarea bind:value={freetext}></textarea>

<h3>Initial split</h3>

{#each totals as total}
  <div>
    {total.name} = {total.sum}
  </div>
{/each}

<hr />

Sum: {total}

<h3>Target per-person value</h3>

Sum: {currencyFormat.format(total / totals.length)}

<h3>Steps</h3>

{#each steps as step}
  <div style="font-weight: bold">
    {step.description}
  </div>
  <br />
{/each}
