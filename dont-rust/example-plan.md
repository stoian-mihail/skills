This is an illustration of a plan's shape and depth. The project, paths and
code are examples. A real plan uses the conventions of its own codebase.

---

# Active plan on the account page

Branch `feature/account-active-plan`, 2026-10-26, mode: task. Short answers,
English.

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

## Map

### 1. Account page render

From `GET /account` to the rendered template. Hops 1c to 1e are new.

- **1a Controller receives the request** · [AccountController::account](src/Controller/AccountController.php#L41)

  Receives the authenticated `User` through the `#[CurrentUser]` attribute.
  Anonymous requests never reach here: the route is under the `ROLE_USER`
  access control in `security.yaml`. Loads the profile and renders the
  template; nothing else is computed in the controller today.

      #[Route('/account', name: 'app_account', methods: ['GET'])]
      public function account(#[CurrentUser] User $user): Response
      {
          $profile = $this->profileService->forUser($user);

          return $this->render('account/account.html.twig', [
              'profile' => $profile,
          ]);
      }

  - **1b Profile service loads the profile** · [ProfileService::forUser](src/Service/ProfileService.php#L28)

    Receives the `User`. Returns a `ProfileView` DTO built from the user and
    its `BillingAddress`. If the user has no billing address the DTO's
    `billingAddress` is null and the template shows an "add address" link.
    Nothing subscription-related passes through here.

        public function forUser(User $user): ProfileView
        {
            $address = $this->addressRepository->findOneBy(['user' => $user, 'type' => AddressType::BILLING]);

            return new ProfileView($user, $address);
        }

    - *`AddressRepository::findOneBy()` (Doctrine)*

  - **1c Subscription service resolves the active plan** · `SubscriptionService::resolveActivePlan` · new

    Receives the user id. Returns the `Plan` of the most recent ACTIVE
    subscription, or null when there is none. See section 1c.

    - **1d Repository fetches the active subscription** · `SubscriptionRepository::findActiveForUser` · new

      Query by user and status, ordered by start date descending, limit one.
      See section 1d.

  - **1e Template renders the plan** · [account.html.twig](templates/account/account.html.twig#L34) · modify

    Receives `profile` and, after this change, `activePlan` (nullable). See
    section 1e.

## Unresolved

- None. `ProfileService` is a concrete class, no interface binding involved.

## Nearby flows not mapped

- Billing address edit: `AccountController::editAddress`, `POST /account/address`.
- Subscription change: `SubscriptionController::change`, which writes the
  rows this feature reads.

## Sections

### AccountController::account — modify · 1a   [pending]
`src/Controller/AccountController.php`

This is where the page is built. The method already loads the profile and
passes it to the template. We add the active plan to the render call so the
template can show it. We get it from `SubscriptionService`, which must be
injected through the constructor since the controller does not use it yet.
The method we call does not exist, so the next section creates it, then we
come back for the template.

Add to the constructor parameters:

    private readonly SubscriptionService $subscriptionService,

Add before the `return $this->render(...)`:

    $activePlan = $this->subscriptionService->resolveActivePlan($user->getId());

Add to the render parameters, after `'profile' => $profile,`:

    'activePlan' => $activePlan,

### SubscriptionService::resolveActivePlan — new · 1c   [pending]
`src/Service/SubscriptionService.php`

Returns the Plan of the user's active subscription, or null when there is
none. The repository does the filtering and ordering; this method only
unwraps the subscription into its plan. Null rather than an exception is
deliberate: the controller and the template treat it as "no active plan".

    public function resolveActivePlan(int $userId): ?Plan
    {
        $subscription = $this->subscriptionRepository->findActiveForUser($userId);

        return $subscription?->getPlan();
    }

### SubscriptionRepository::findActiveForUser — new · 1d   [pending]
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

### account.html.twig — modify · 1e   [pending]
`templates/account/account.html.twig`

Shows the plan under the billing address. `activePlan` comes from the
controller and may be null, so both cases are handled in the template and
the page never breaks for a user without a subscription.

Add after the `{% block billing %}` section:

    <section class="plan">
        <h2>Your plan</h2>
        {% if activePlan %}
            <p>{{ activePlan.name }} · renews {{ activePlan.renewsAt|date('j M Y') }}</p>
        {% else %}
            <p>No active plan</p>
        {% endif %}
    </section>
