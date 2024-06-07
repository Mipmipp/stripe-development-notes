# Stripe Connect Documentation
This guide provides a basic introduction to Stripe Connect, referencing the [official Stripe documentation](https://docs.stripe.com/connect).  Remember, it's important to consult the official documentation for the most up-to-date information and detailed instructions.

## Onboarding
Stripe Connect offers three methods for onboarding connected accounts:

- **Stripe-Hosted Onboarding:** This method utilizes a Stripe-hosted form to simplify the onboarding process for your users. [Documentation](https://docs.stripe.com/connect/custom/hosted-onboarding)
- **Embedded Onboarding:** For a more customized user experience, you can embed Stripe's onboarding components directly within your application. [Documentation](https://docs.stripe.com/connect/embedded-onboarding)
- **API Onboarding:** For maximum control, you can leverage Stripe's API to manage the onboarding process entirely within your application. [Documentation](https://docs.stripe.com/connect/api-onboarding)

---
> This guide will focus on Stripe-Hosted Onboarding as a starting point.
---

**Steps:**
1) **Customize Onboarding Form:** Stripe allows you to personalize the onboarding form used for your connected accounts. Through your [Stripe dashboard settings](https://dashboard.stripe.com/account/applications/settings), you can update the form's appearance with your company logo, colors, and other branding elements.  This helps create a seamless user experience by maintaining consistency between your platform and the onboarding process.
2) **Create a Connected Account:** You'll likely create an API method to handle user account creation. This method will interact with Stripe's API to establish a connected account for each new user. Here's a NestJS code example that demonstrates this process:
``` typescript
async createPayoutAccount(username: string): Promise<string> {
  const response = await this.stripe.accounts.create({
    type: 'express', // https://docs.stripe.com/api/accounts/create#create_account-type
    email: username,
    capabilities: {
      transfers: { requested: true },
    },
  });

  return response.id; // (property) Stripe.Account.id: string
}
```
3) **Determine Onboarding Information:** Stripe's hosted onboarding offers flexibility in the information you collect from your connected accounts. There are two primary categories for required information:
- `currently_due`: This information is essential for the connected account to function immediately. It typically includes basic details like the account holder's email address.
- `eventually_due`: This category encompasses information that may be required at a later date, depending on specific business needs or regulations. For example, tax information might fall under eventually_due.
4) **Create an Account Link:** Once you've established a connected account, you'll need to create an Account Link using Stripe's API.  This link serves as the URL that directs your user to Stripe's hosted onboarding flow. Here's a Nestjs code example demonstrating this process:
  ``` typescript
  async createPayoutOnboardingLink(accId: string): Promise<string> {
    const response = await this.stripe.accountLinks.create({
      account: accId,
      refresh_url: this.configService.get('stripe.refreshUrl'),
      return_url: this.configService.get('stripe.returnUrl'),
      type: 'account_onboarding', // https://docs.stripe.com/api/account_links/create#create_account_link-type
      collect: 'eventually_due', // https://docs.stripe.com/api/account_links/create#create_account_link-collection_options-fields
    });
  
    return response.url; // (property) Stripe.AccountLink.url: string
  }
  ```
**Important Considerations:**
- **Environment Variables:** The `refresh_url` and `return_url` parameters within the `createAccountLink` method should reference environment variables configured within your application. This ensures you're using the appropriate URLs for your development, testing, and production environments.
  - `refresh_url`: This URL should point to a method on your server that can regenerate an Account Link with the same parameters in case the initial link expires. This helps prevent onboarding disruptions.
  - `return_url`This URL specifies where your user will be redirected after completing (or abandoning) the onboarding process. This allows you to determine the account's onboarding status (as described in step 5).
- **Redirection:** It's important to redirect the user directly to the provided Account Link URL. Avoid displaying or logging the URL itself for security reasons.
5) **Handle New Requirements:** Stripe may identify additional verification requirements for a connected account even after the initial onboarding process. These `eventually_due` requirements can prevent account functionality until fulfilled. Here's how to address this scenario:
- **Webhooks (Optional):** While not covered in this guide, Stripe offers webhooks that can notify you when new requirements become due for a connected account. You can find more information about webhooks and verification in the Stripe documentation (https://docs.stripe.com/connect/handling-api-verification#verification-process, https://docs.stripe.com/connect/webhooks).
- **Manual Verification Check:** Alternatively, you can implement a manual check within your application to verify if any new requirements exist for a connected account. Here's an example TypeScript code snippet demonstrating how to retrieve an account's requirements using Stripe's API:
  ``` typescript
  async getAccountRequeriments(
    accId: string,
  ): Promise<Stripe.Account.Requirements> {
    const response = await this.stripe.accounts.retrieve(accId);
  
    return response.requirements; // (property) Stripe.Account.requirements?: Stripe.Account.Requirements
  }
  ```
  Examine the `response.requirements.disabled_reason` property within the retrieved data. An empty array (`[]`) for `disabled_reason` indicates no outstanding requirements.
  - **Conditional Redirection:** Based on the presence of requirements:
    - **No Requirements:** If `disabled_reason` is an empty array (`[]`), the account has met Stripe's verification requirements. In this case, you can:
      - **Get Onboarding Link:** Utilize the previously described logic from step 4 (using `createPayoutOnboardingLink`) to generate an Account Link URL for the connected account.
      - **Redirect User:** Directly redirect the user to the retrieved Account Link URL using a browser redirect. **Avoid displaying or logging the URL itself for security reasons.** This sends the user to Stripe's dashboard where they can access their account information and manage any optional details.
    - **Requirements Present:** If `disabled_reason` contains any elements (indicating requirements), new requirements exist. You can:
      - Inform the user about the necessary steps.
      - Guide them through the verification process within Stripe's dashboard.
      - Adjust your application flow to restrict actions until verification is complete.
6) **Generate Login Link:** Stripe's API allows you to create temporary login links for connected accounts.  These links grant the account holder one-time access to their Stripe dashboard (Express Dashboard specifically). Here's how to implement this functionality:
``` typescript
async createPayoutLoginLink(accId: string): Promise<string> {
  const response = await this.stripe.accounts.createLoginLink(accId);

  return response.url; // (property) Stripe.LoginLink.url: string
}
```
7) **Handle User-Initiated Updates:** There may be scenarios where a user needs to update their information associated with their connected account. Stripe's hosted onboarding functionality doesn't directly support user-initiated updates within the onboarding flow itself. However, you can leverage a combination of redirects and Account Links to achieve this:
- **Dedicated Update Page:** Create a dedicated page within your application specifically for handling user-initiated updates. In your example, this page is referred to as `/ourPageThatHandleStep8/update`.
- **Triggering the Update:** Within your application's user interface, provide a mechanism for users to initiate the update process. This could be a button, link, or menu option that directs the user to your dedicated update page.
- **Redirect and Logic:**  When a user lands on your update page (e.g., `/ourPageThatHandleStep8/update`), you can perform the following actions:
  - **Indicate Update (Optional):** You can include a flag (e.g., the `update` parameter in your example) within the URL to signal that the user intends to update their information. While not strictly necessary for Stripe's API, this can help with internal logic within your application.
  - **Regardless of Requirements:** Unlike step 5, you don't need to check for outstanding requirements here. The goal is to allow the user to access the onboarding flow for updates even if new requirements exist.
  - **Generate Account Link:** Utilize the previously described logic from step 4 (using `createPayoutOnboardingLink`) to generate an Account Link URL for the connected account.
- **Redirect to Onboarding:** Directly redirect the user to the retrieved Account Link URL using a browser redirect.  **Avoid displaying or logging the URL itself for security reasons.** This sends the user to Stripe's onboarding flow where they can update their information as needed.
8) **React Example with React Query and TypeScript:** This example demonstrates how to integrate the concepts from previous steps into a React application using React Query for data fetching and management.
``` typescript
enum StripeConnectRedirectHandlerPageTypes {
	UPDATE = 'update',
	REFRESH = 'refresh',
	RETURN = 'return',
}

export const StripeConnectRedirectHandler = (): JSX.Element => {
	const navigate = useNavigate();
	const { action } = useParams() as {
		action: StripeConnectRedirectHandlerPageTypes;
	};

	const {
		data: accountRequirements,
		isLoading,
		isError,
	} = useGetAccountRequirements(); // get user account requirements (this inside use user's stripe connect account's id saved in localstorage)

	const { mutate: redirectToOnboardingLink } = useGetOnboardingLink(navigate); // function that fetch onboardingLink and redirect user to it
	const { mutate: redirectToDashboardExpressLink } =
		useGetDashboardExpressLink(navigate); // function that fetch express dashboard link and redirect user to it

	useEffect(() => {
		const dataIsReady = !isLoading && !isError;

		const handleAccountRequirements = () => {
      // check if user is disabled
			const accountIsDisabled =
				accountRequirements?.disabled_reason &&
				accountRequirements?.disabled_reason.length > 0;

      // should be redirecte to onboarding link if stripe return to this page with refresh params (refresh_url), if we redirect user to this page with update params (update) or if account have requirements (is disabled)
			const shouldRedirectToOnboardingLink =
				action === onboardginPageTypes.REFRESH ||
				action === onboardginPageTypes.UPDATE ||
				accountIsDisabled;

			if (shouldRedirectToOnboardingLink) {
				redirectToOnboardingLink();
			} else {
        // if account is not disable, just redirect user to their dashboard express
				redirectToDashboardExpressLink();
			}
		};

		if (dataIsReady) {
			handleAccountRequirements();
		}
	}, [
		accountRequirements,
		isLoading,
		isError,
		action,
		redirectToOnboardingLink,
		redirectToDashboardExpressLink,
	]);

	if (isError) {
		const ERROR_MESSAGE = 'Error getting your account requirements';
		const HOME_PAGE = '/';

		errorToast(ERROR_MESSAGE);
		navigate(HOME_PAGE);
	}

	return <BasicLoading />;
};

```
9) Flow diagram



