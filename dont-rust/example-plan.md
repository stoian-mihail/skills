This is an illustration of a plan's shape and level of detail. The project,
language, framework and code are examples. A real plan uses the conventions
of the codebase it is written for.

---

# Active plan on the account page

## Goal

The account page shows the user's profile and billing address but nothing
about their subscription. Users keep opening support tickets to ask which
plan they are on. We want the account page to show the name of the active
plan and its renewal date.

A user can have several rows in the subscription table over time, since every
upgrade and downgrade inserts a new row. Only one row is ACTIVE at a time,
but there have been data glitches with two ACTIVE rows, so we take the most
recent by start date rather than assuming uniqueness. If there is no active
subscription, the page shows "No active plan" and never errors.

## AccountController::account — modify   [pending]
`src/Controller/AccountController.php`

This is where the page is built. The method already loads the profile and
passes it to the template. We add the active plan to the render call so the
template can show it. We get it from `SubscriptionService`, which needs to be
injected through the constructor since the controller does not use it yet.
The method we call does not exist, so the next section creates it, then we
come back for the template.

Add to the constructor parameters:

    private readonly SubscriptionService $subscriptionService,

Add before the `return $this->render(...)`:

    $activePlan = $this->subscriptionService->resolveActivePlan($user->getId());

Add to the render parameters, after `'profile' => $profile,`:

    'activePlan' => $activePlan,

## SubscriptionService::resolveActivePlan — new   [pending]
`src/Service/SubscriptionService.php`

Returns the Plan of the user's active subscription, or null when there is
none. The repository does the filtering and ordering; this method just
unwraps the subscription into its plan. Null rather than an exception is
deliberate, the controller and template treat it as "no active plan".

    public function resolveActivePlan(int $userId): ?Plan
    {
        $subscription = $this->subscriptionRepository->findActiveForUser($userId);

        return $subscription?->getPlan();
    }

## SubscriptionRepository::findActiveForUser — new   [pending]
`src/Repository/SubscriptionRepository.php`

Fetches the most recent ACTIVE subscription for a user. Ordering by start
date descending with a limit of one is what protects us from the duplicate
ACTIVE rows mentioned in the goal.

    public function findActiveForUser(int $userId): ?Subscription
    {
        return $this->createQueryBuilder('s')
            ->andWhere('s.user = :userId')
            ->andWhere('s.status = :status')
            ->setParameter('userId', $userId)
            ->setParameter('status', SubscriptionStatus::ACTIVE)
            ->orderBy('s.startDate', 'DESC')
            ->setMaxResults(1)
            ->getQuery()
            ->getOneOrNullResult();
    }

## account.html.twig   [pending]
`templates/account/account.html.twig`

Shows the plan under the billing address. The `activePlan` variable comes
from the controller and may be null, so both cases are handled in the
template and the page never breaks for a user without a subscription.

Add after the `{% block billing %}` section:

    <section class="plan">
        <h2>Your plan</h2>
        {% if activePlan %}
            <p>{{ activePlan.name }} · renews {{ activePlan.renewsAt|date('j M Y') }}</p>
        {% else %}
            <p>No active plan</p>
        {% endif %}
    </section>
