# Raydium launchpad program upgrade with new features

- Support creator fee;
- Support token_2022 for base token;
- Support platform fee claimed from `platform_fee_vault`;

## How

1. Add `creator_fee_rate`， `transfer_fee_extension_auth` and `curve_params` fields in `PlatformConfig` account layout;

    ```rust
    #[account]
    pub struct PlatformConfig {
        /// The epoch for update interval
        pub epoch: u64,
        /// The platform fee wallet
        pub platform_fee_wallet: Pubkey,
        /// The platform nft wallet to receive the platform NFT after migration if platform_scale is not 0(Only support MigrateType::CPSWAP)
        pub platform_nft_wallet: Pubkey,
        /// Scale of the platform liquidity quantity rights will be converted into NFT(Only support MigrateType::CPSWAP)
        pub platform_scale: u64,
        /// Scale of the token creator liquidity quantity rights will be converted into NFT(Only support MigrateType::CPSWAP)
        pub creator_scale: u64,
        /// Scale of liquidity directly to burn
        pub burn_scale: u64,
        /// The platform fee rate
        pub fee_rate: u64,
        /// The platform name
        pub name: [u8; NAME_SIZE],
        /// The platform website
        pub web: [u8; WEB_SIZE],
        /// The platform img link
        pub img: [u8; IMG_SIZE],
        /// The platform specifies the trade fee rate after migration to cp swap
        pub cpswap_config: Pubkey,
        /// Creator fee rate
        pub creator_fee_rate: u64,
        /// If the base token belongs to token2022, then you can choose to support the transferfeeConfig extension, which includes permissions such as `transfer_fee_config_authority`` and `withdraw_withheld_authority`.
        /// When initializing mint, `withdraw_withheld_authority` and `transfer_fee_config_authority` both belongs to the contract.
        /// Once the token is migrated to AMM, the authorities will be reset to this value
        pub transfer_fee_extension_auth: Pubkey,
         /// padding for future updates
        pub padding: [u8; 180],
        /// The parameters for launching the pool
        pub curve_params: Vec<PlatformCurveParam>,
    }

    pub struct PlatformCurveParam {
        /// The epoch for update interval, 0 means not update
        pub epoch: u64,
        /// The curve params index
        pub index: u8,
        /// The global config address
        pub global_config: Pubkey,
        /// bonding curve param
        pub bonding_curve_param: BondingCurveParam,
        /// padding for future updates
        pub padding: [u64; 50],
    }


    pub struct BondingCurveParam {
        // curve params
        /// Migrate to AMM or CpSwap, 0: amm， 1: cpswap，
        /// Neither 0 nor 1: invalid
        pub migrate_type: u8,
        /// The migrate fee on, 0 means fee on the quote token, 1 means fee on both token
        /// Neither 0 nor 1: invalid
        pub migrate_cpmm_fee_on: u8,
        /// The supply of the token,
        /// 0: invalid
        pub supply: u64,
        /// The total base sell of the token
        /// 0: invalid
        pub total_base_sell: u64,
        /// The total quote fund raising of the token
        /// 0: invalid
        pub total_quote_fund_raising: u64,
        // vesting params
        /// total amount of tokens to be unlocked
        /// u64::MAX: invalid
        pub total_locked_amount: u64,
        /// Waiting time in seconds before unlocking after fundraising ends
        /// u64::MAX: invalid
        pub cliff_period: u64,
        /// Unlocking period in seconds
        /// u64::MAX: invalid
        pub unlock_period: u64,
    }
    ```

1. Add `creator_fee_rate` field in `PlatformParams` for the `create_platform_config` instruction params data;

    When the platform creates(`create_platform_config` instruction) or updates(`update_platform_config` instrucion) `PlatformConfig` accounts, the `creator_fee_rate` needs to be passed in.

2. Add account `transfer_fee_extension_authority` for `create_platform_config` instruction.

    ```rust
    #[derive(Accounts)]
    pub struct CreatePlatformConfig<'info> {
        /// The account paying for the initialization costs  
        #[account(mut)]
        pub platform_admin: Signer<'info>,

        /// CHECK:The wallet for the platform to receive platform fee
        #[account()]
        pub platform_fee_wallet: UncheckedAccount<'info>,

        /// CHECK:The wallet for the platform to receive migrate liquidity nft(Only support cpswap program)
        #[account()]
        pub platform_nft_wallet: UncheckedAccount<'info>,

        /// The platform config account
        #[account(
            init,
            seeds =[
                PLATFORM_CONFIG_SEED.as_bytes(),
                platform_admin.key().as_ref(),
            ],
            bump,
            payer = platform_admin,
            space = PlatformConfig::LEN,
        )]
        pub platform_config: Account<'info, PlatformConfig>,

        /// CHECK:CpSwap config Account.
        #[account(
            owner = crate::cpswap_program::ID,
        )]
        pub cpswap_config: UncheckedAccount<'info>,

        /// Required for account creation
        pub system_program: Program<'info, System>,

        /// CHECK: If you support the creation of token2022's token and the extension of transferFeeConfig, please set this account.
        #[account()]
        pub transfer_fee_extension_authority: UncheckedAccount<'info>,
    }
    ```

3. Add the required accounts(`system_program`, `platform_fee_vault`, `creator_fee_vault`) in `remaining_accounts` with `buy_exact_in`, `buy_exact_out`, `sell_exact_in`, `sell_exact_out` instructions

4. Add `claim_creator_fee` and `claim_platform_fee_from_vault` instructions to support claim fees.

5. Add `initialize_with_token_2022` instruction to initialize token22 base token with `TransferFeeConfig` extension and `AmmFeeOn`, for details, please refer to the IDL.

6. Add `initialize_v2` instruction to initialize spl-token base token, compared to `initialize`, the `AmmFeeOn` parameter has been added to specify the mode of creator fee.

7. Add `update_platform_curve_param` instruction to update/add a Platform `BondingCurveParam`.

8. Add `remove_platform_curve_param` instruction to delate a Platform `BondingCurveParam`.

## What need you to do

1. You must add accounts in `remaining_accounts` for `buy_exact_in`, `buy_exact_out`, `sell_exact_in`, `sell_exact_out` instructions as follows:

    This following update must be completed before the on-chain `unix_timestamp`(Hard code in program) specified.\
    Updates are compatible before the specified timestamp, the contract will enforce checking of `remaining_accounts` according to the following rules after that.

    ```rust
    let platform_fee_vault = Pubkey::find_program_address(
            &[platform_config.key().as_ref(), quote_token_mint.key().as_ref()],
            &launch_program_id,
        ).0;
    let creator_fee_vault = Pubkey::find_program_address(
            &[pool_creator.as_ref(), quote_token_mint.key().as_ref()],
            &launch_program_id,
        ).0;

    let remaining_accounts = if share_fee_rate > 0 {
        vec![
        AccountMeta::new(fee_share_receiver,false),
        AccountMeta::new_readonly(system_program::id(),false),
        AccountMeta::new(platform_fee_vault,false),
        AccountMeta::new(creator_fee_vault,false),
        ]
    }
    else {
        vec![
        AccountMeta::new_readonly(system_program::id(),false),
        AccountMeta::new(platform_fee_vault,false),
        AccountMeta::new(creator_fee_vault,false),
        ]
    }
    ```

2. If you have already created your `PlatformConfig` account

    - You can use `update_platform_config` instruction to update the `creator_fee_rate` and `transfer_fee_extension_auth` fields;
    - You can claim platform fees by `claim_creator_fee`(claim fees from `platform_fee` saved `PoolState` is not 0) and `claim_platform_fee_from_vault`(claim from `platform_fee_vault`) instructions;
    - You can add `BondingCurveParam` to restrict the use of all tokens created with your platform config
  
3. If you have not created your `PlatformConfig` account

    You can refer to the new idl attached to create it.

4. If you want to create token_2022 base token, you can use `initialize_with_token_2022` instruction.

5. Quickly integrate the `initializeV2` instruction; After 2025-08-21 13:00:00 UTC, the `initialize` instruction will be deprecated.
