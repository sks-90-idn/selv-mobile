<script>
    import { Plugins, KeyboardInfo } from '@capacitor/core';
    import { onDestroy } from 'svelte';
    import { flip } from 'svelte/animate';

    import Button from '~/components/Button';
    import TextField from '~/components/TextField';
    import Header from '~/components/Header';

    import { preparePersonalInformation, getRandomUserData, goto, delay, generateRandomId } from '~/lib/helpers';
    import { error, hasSetupAccount, storedCredentials, account } from '~/lib/store';
    import { createIdentity, storeIdentity, retrieveIdentity, createSelfSignedCredential } from '~/lib/identity';
    import { SchemaNames } from '~/lib/identity/schemas';
    import { __WEB__ } from '~/lib/platform';

    const { Keyboard } = Plugins;
    let isCreatingCredentials = false;
    let firstName = '';
    let isKeyboardActive = false;

    let background;
    let keyboardHeight;

    const isRunningOnMobile = !__WEB__;

    if (isRunningOnMobile) {
        Keyboard.addListener('keyboardWillShow', (info) => {
            keyboardHeight = info.keyboardHeight;
            isKeyboardActive = true;
        });

        Keyboard.addListener('keyboardWillHide', () => {
            isKeyboardActive = false;
        });
    }

    function handleOuterClick() {
        if (event.target === background) {
            event.preventDefault();

            if (document.activeElement) {
                document.activeElement.blur();
            }
        }
    }

    function save() {
        if (isCreatingCredentials) {
            return;
        }
        isCreatingCredentials = true;

        if (isRunningOnMobile) {
            Keyboard.hide();
        }

        setTimeout(() => {
            // Hide the error notification (if any)
            error.set(null);

            account.set({ name: firstName });

            retrieveIdentity()
                .then((identity) =>
                    identity
                        ? Promise.resolve(identity)
                        : Promise.race([
                              createIdentity(),
                              new Promise((resolve, reject) => {
                                  setTimeout(() => reject(new Error('Error creating identity')), 20000);
                              }),
                          ]).then((newIdentity : Identity) => storeIdentity('did', newIdentity).then(() => newIdentity))
                )
                .then((identity) => {
                    delay(2000);
                    return getRandomUserData().then((data) =>
                        Promise.all([
                            createSelfSignedCredential(identity, SchemaNames.ADDRESS, {
                                UserAddress: {
                                    City: data.location.city,
                                    State: data.location.state,
                                    Country: data.location.country,
                                    Postcode: data.location.postcode.toString(),
                                    Street: data.location.street.number.toString(),
                                    House: data.location.street.name,
                                },
                            }),
                            createSelfSignedCredential(identity, SchemaNames.PERSONAL_DATA, {
                                UserPersonalData: {
                                    UserName: {
                                        FirstName: firstName,
                                        LastName: data.name.last,
                                    },
                                    UserDOB: {
                                        Date: new Date(data.dob.date).toDateString(),
                                    },
                                    Birthplace: data.location.city,
                                    Nationality: data.location.country,
                                    IdentityCardNumber: data.id.value,
                                    PassportNumber: Math.random().toString(36).substring(7).toUpperCase(),
                                },
                            }),
                            createSelfSignedCredential(identity, SchemaNames.CONTACT_DETAILS, {
                                UserContacts: {
                                    Email: data.email,
                                    Phone: data.phone,
                                },
                            }),
                        ])
                    );
                })
                .then((result) => {
                    const [addressCredential, personalDataCredential, contactDetailsCredential] = result;

                    storedCredentials.update((prev) =>
                        [...prev, ...[addressCredential, personalDataCredential, contactDetailsCredential]].map((credential) => ({
                            credentialDocument: { ...credential },
                            metaInformation: { issuer: 'selv' },
                            id: generateRandomId()
                        }))
                    );

                    isCreatingCredentials = false;
                    hasSetupAccount.set(true);
                    goto('onboarding/home');
                })
                .catch((err) => {
                    error.set('Error creating identity. Please try again.');

                    isCreatingCredentials = false;
                });
        }, 500);
    }
</script>

<style>
    main {
        height: 100%;
        background-color: var(--bg);
        display: flex;
        flex-direction: column;
        justify-content: space-between;
        overflow-y: auto;
        -webkit-overflow-scrolling: touch;
        position: absolute;
        padding: 6vh 5vw;
    }

    .content {
        text-align: center;
        z-index: 1;
    }

    .content > img {
        mix-blend-mode: multiply;
    }

    footer {
        padding: 0px 7vw;
    }

    img {
        width: 27vh;
        height: 27vh;
    }

    .info {
        font-family: 'Inter', sans-serif;
        font-style: normal;
        font-weight: normal;
        font-size: 2.08vh;
        line-height: 3.3vh;
        color: #6f7a8d;
        text-align: center;
        padding: 0px 3vw;
        width: 100%;
    }
</style>

{#each [true] as item, index (item)}
    <main
        bind:this="{background}"
        on:click="{handleOuterClick}"
        style="top: {isKeyboardActive ? `-${keyboardHeight}px` : '0'}"
        animate:flip="{{ duration: 350 }}"
    >
        <Header text="Set your first name" />

        <div class="content"><img src="set-name.png" alt="" /></div>

        <p class="info">Selv will generate you an identity using randomised personal information.</p>

        <TextField disabled="{isCreatingCredentials}" bind:value="{firstName}" placeholder="First name" />

        <footer>
            <Button
                loading="{isCreatingCredentials}"
                loadingText="{'Generating identity'}"
                disabled="{firstName.length === 0}"
                label="Save Name"
                onClick="{save}"
            />
        </footer>
    </main>
{/each}
