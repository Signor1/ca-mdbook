# Cairo Academy: Understanding a Basic Lottery Contract

This chapter delves into the basics of a lottery contract implemented in Cairo. This contract demonstrates the core functionalities of a basic lottery contract, providing a practical example for learning how to build a basic lottery contract on StarkNet.

## Purpose and Functionality

The provided Cairo code defines a basic lottery contract. It implements essential functionalities such as:

- **Start Lottery:** This starts the operation of a lottery round. This takes in parameters:
ticket_price : Price of a single lottery ticket in wei
max_tickets : Maximum number of tickets for this lottery round
prize_distribution : Array containing percentage distribution of prizes (must sum to 100)
duration : Duration of the lottery in seconds

- **Buy lottery tickets:** Buy lottery tickets. This takes parameters:
ticket_count : Number of tickets to purchase

- **End the lottery and select winners:**. This takes parametrs:
random_seed : Additional entropy source to enhance randomness

- **Claim prizes for the caller if they won**

We have some view functions. These are functions that read data from the blockcahin.

- **Get lottery information** : This get the lottery information
- **Get ticket holder** : This gets the information about the lottery ticket holder
- **Get Winner** : This gets the winner information
- **Get user tickets** : This gets information about the user tickets
- **Is the lottery active** This return a bool (true or flase) the active state of the lottery ticket

This contract serves as a foundational example for understanding the mechanics of Lottery Contract within the StarkNet ecosystem.

``` rust
use starknet::ContractAddress;
use array::ArrayTrait;
use option::OptionTrait;

#[starknet::interface]
pub trait ILotteryContract<TContractState> {
    fn start_lottery(
        ref self: TContractState,
        ticket_price: u256,
        max_tickets: u32,
        prize_distribution: Array<u8>,
        duration: u64
    );
    fn buy_tickets(ref self: TContractState, ticket_count: u32);
    fn end_lottery(ref self: TContractState, random_seed: felt252);
    fn claim_prize(ref self: TContractState);
    fn get_lottery_info(self: @TContractState) -> LotteryInfo;
    fn get_ticket_holders(self: @TContractState) -> Array<ContractAddress>;
    fn get_winners(self: @TContractState) -> Array<Winner>;
    fn get_user_tickets(self: @TContractState, user: ContractAddress) -> u32;
    fn is_lottery_active(self: @TContractState) -> bool;
}

#[derive(Copy, Drop, Serde, starknet::Store)]
pub struct LotteryInfo {
    lottery_id: u32,
    ticket_price: u256,
    max_tickets: u32,
    tickets_sold: u32,
    prize_pool: u256,
    start_time: u64,
    end_time: u64,
    lottery_status: u8, 
}

#[derive(Copy, Drop, Serde)]
pub struct Winner {
    address: ContractAddress,
    prize_amount: u256,
    claimed: bool
}

mod Errors {
    pub const LOTTERY_ALREADY_ACTIVE: felt252 = 'Lottery already active';
    pub const INVALID_TICKET_PRICE: felt252 = 'Invalid ticket price';
    pub const INVALID_MAX_TICKETS: felt252 = 'Invalid max tickets';
    pub const INVALID_DURATION: felt252 = 'Invalid duration';
    pub const INVALID_DISTRIBUTION: felt252 = 'Invalid prize distribution';
    pub const LOTTERY_NOT_ACTIVE: felt252 = 'Lottery not active';
    pub const LOTTERY_NOT_ENDED: felt252 = 'Lottery still active';
    pub const LOTTERY_ALREADY_ENDED: felt252 = 'Lottery already ended';
    pub const MAX_TICKETS_REACHED: felt252 = 'Max tickets reached';
    pub const INSUFFICIENT_PAYMENT: felt252 = 'Insufficient payment';
    pub const NOT_OWNER: felt252 = 'Not the contract owner';
    pub const NO_PRIZES_TO_CLAIM: felt252 = 'No prizes to claim';
    pub const PRIZE_ALREADY_CLAIMED: felt252 = 'Prize already claimed';
}

#[starknet::contract]
pub mod LotteryContract {
    use super::{
        ILotteryContract, LotteryInfo, Winner, Errors,
        ArrayTrait, ContractAddress, OptionTrait
    };
    use core::num::traits::Zero;
    use starknet::{
        get_caller_address, get_block_timestamp, get_block_number, get_tx_info,
        get_contract_address
    };
    use openzeppelin_token::erc20::interface::{IERC20Dispatcher, IERC20DispatcherTrait};
    use core::hash::{LegacyHash, HashStateTrait};
    use core::pedersen::PedersenTrait;
    use core::traits::Into;
    use core::array::SpanTrait;

    #[storage]
    struct Storage {
        owner: ContractAddress,
        payment_token: IERC20Dispatcher,
        lottery_id: u32,
        active_lottery: LotteryInfo,
        ticket_holders: LegacyMap::<u32, ContractAddress>, 
        user_tickets: LegacyMap::<(u32, ContractAddress), u32>, count
        winners: LegacyMap::<(u32, u32), Winner>, 
        winner_count: LegacyMap::<u32, u32>, 
        prize_distribution: LegacyMap::<(u32, u32), u8>, 
        prize_distribution_count: LegacyMap::<u32, u32>, 
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        LotteryStarted: LotteryStarted,
        TicketsPurchased: TicketsPurchased,
        LotteryEnded: LotteryEnded,
        PrizeClaimed: PrizeClaimed,
    }

    #[derive(Drop, starknet::Event)]
    struct LotteryStarted {
        lottery_id: u32,
        ticket_price: u256,
        max_tickets: u32,
        duration: u64,
    }

    #[derive(Drop, starknet::Event)]
    struct TicketsPurchased {
        lottery_id: u32,
        buyer: ContractAddress,
        ticket_count: u32,
        total_cost: u256,
    }

    #[derive(Drop, starknet::Event)]
    struct LotteryEnded {
        lottery_id: u32,
        total_tickets_sold: u32,
        prize_pool: u256,
        timestamp: u64,
    }

    #[derive(Drop, starknet::Event)]
    struct PrizeClaimed {
        lottery_id: u32,
        winner: ContractAddress,
        prize_amount: u256,
    }

    #[constructor]
    fn constructor(
        ref self: ContractState,
        payment_token_address: ContractAddress,
    ) {
        self.owner.write(get_caller_address());
        self.payment_token.write(IERC20Dispatcher { contract_address: payment_token_address });
        self.lottery_id.write(0);
    }

    #[abi(embed_v0)]
    impl LotteryContractImpl of super::ILotteryContract<ContractState> {
        fn start_lottery(
            ref self: ContractState, 
            ticket_price: u256, 
            max_tickets: u32,
            prize_distribution: Array<u8>,
            duration: u64
        ) {
            self.assert_only_owner();
            
            assert(!self.is_lottery_active(@self), Errors::LOTTERY_ALREADY_ACTIVE);
            
            assert(ticket_price > 0, Errors::INVALID_TICKET_PRICE);
            assert(max_tickets > 0, Errors::INVALID_MAX_TICKETS);
            assert(duration > 0, Errors::INVALID_DURATION);
            
            let mut total_percentage: u8 = 0;
            let mut i: u32 = 0;
            let distribution_len = prize_distribution.len();
            assert(distribution_len > 0, Errors::INVALID_DISTRIBUTION);

            let distribution_span = prize_distribution.span();
            loop {
                if i >= distribution_len {
                    break;
                }
                let percentage = *distribution_span.at(i);
                total_percentage += percentage;
                self.prize_distribution.write((self.lottery_id.read() + 1, i), percentage);
                i += 1;
            }
            
            assert(total_percentage == 100, Errors::INVALID_DISTRIBUTION);
            self.prize_distribution_count.write(self.lottery_id.read() + 1, distribution_len);
            
            let new_lottery_id = self.lottery_id.read() + 1;
            let start_time = get_block_timestamp();
            let end_time = start_time + duration;
            
            self.active_lottery.write(LotteryInfo {
                lottery_id: new_lottery_id,
                ticket_price,
                max_tickets,
                tickets_sold: 0,
                prize_pool: 0,
                start_time,
                end_time,
                lottery_status: 1,
            });
            
            self.lottery_id.write(new_lottery_id);
            
            self.emit(LotteryStarted {
                lottery_id: new_lottery_id,
                ticket_price,
                max_tickets,
                duration,
            });
        }

        fn buy_tickets(ref self: ContractState, ticket_count: u32) {
            assert(self.is_lottery_active(@self), Errors::LOTTERY_NOT_ACTIVE);
            
            let caller = get_caller_address();
            let active_lottery = self.active_lottery.read();
            
            assert(active_lottery.tickets_sold + ticket_count <= active_lottery.max_tickets, Errors::MAX_TICKETS_REACHED);
            
            let total_cost = active_lottery.ticket_price * ticket_count.into();
            
            self.payment_token.read().transfer_from(
                caller, 
                get_contract_address(), 
                total_cost
            );
            
            let current_tickets_sold = active_lottery.tickets_sold;
        
            let mut i: u32 = 0;
            loop {
                if i >= ticket_count {
                    break;
                }
                self.ticket_holders.write(current_tickets_sold + i, caller);
                i += 1;
            }
            
            let user_current_tickets = self.user_tickets.read((active_lottery.lottery_id, caller));
            self.user_tickets.write((active_lottery.lottery_id, caller), user_current_tickets + ticket_count);
            
            self.active_lottery.write(LotteryInfo {
                lottery_id: active_lottery.lottery_id,
                ticket_price: active_lottery.ticket_price,
                max_tickets: active_lottery.max_tickets,
                tickets_sold: active_lottery.tickets_sold + ticket_count,
                prize_pool: active_lottery.prize_pool + total_cost,
                start_time: active_lottery.start_time,
                end_time: active_lottery.end_time,
                lottery_status: active_lottery.lottery_status,
            });
            
            self.emit(TicketsPurchased {
                lottery_id: active_lottery.lottery_id,
                buyer: caller,
                ticket_count,
                total_cost,
            });
        }

        fn end_lottery(ref self: ContractState, random_seed: felt252) {
            self.assert_only_owner();
            
            assert(self.is_lottery_active(@self), Errors::LOTTERY_NOT_ACTIVE);
            
            let active_lottery = self.active_lottery.read();
        
            assert(
                get_block_timestamp() >= active_lottery.end_time || 
                active_lottery.tickets_sold == active_lottery.max_tickets,
                Errors::LOTTERY_NOT_ENDED
            );
            
            if active_lottery.tickets_sold == 0 {
                self.active_lottery.write(LotteryInfo {
                    lottery_id: active_lottery.lottery_id,
                    ticket_price: active_lottery.ticket_price,
                    max_tickets: active_lottery.max_tickets,
                    tickets_sold: active_lottery.tickets_sold,
                    prize_pool: active_lottery.prize_pool,
                    start_time: active_lottery.start_time,
                    end_time: active_lottery.end_time,
                    lottery_status: 2, 
                });
                return;
            }

            let random_value = self.generate_random_value(random_seed);
            
            let prize_pool = active_lottery.prize_pool;
            let mut winner_count: u32 = 0;
            let distribution_count = self.prize_distribution_count.read(active_lottery.lottery_id);
            
            let mut i: u32 = 0;
            loop {
                if i >= distribution_count {
                    break;
                }
                
                let percentage = self.prize_distribution.read((active_lottery.lottery_id, i));
                let prize_amount = (prize_pool * percentage.into()) / 100_u256;
                
                if prize_amount > 0 {
                    let winner_ticket = self.select_random_ticket(
                        active_lottery.tickets_sold, 
                        random_value, 
                        i
                    );
                    let winner_address = self.ticket_holders.read(winner_ticket);
                    
                    self.winners.write(
                        (active_lottery.lottery_id, i),
                        Winner { 
                            address: winner_address, 
                            prize_amount,
                            claimed: false 
                        }
                    );
                    
                    winner_count += 1;
                }
                
                i += 1;
            }
            
            self.winner_count.write(active_lottery.lottery_id, winner_count);
            
            self.active_lottery.write(LotteryInfo {
                lottery_id: active_lottery.lottery_id,
                ticket_price: active_lottery.ticket_price,
                max_tickets: active_lottery.max_tickets,
                tickets_sold: active_lottery.tickets_sold,
                prize_pool: active_lottery.prize_pool,
                start_time: active_lottery.start_time,
                end_time: active_lottery.end_time,
                lottery_status: 2, 
            });
            
            self.emit(LotteryEnded {
                lottery_id: active_lottery.lottery_id,
                total_tickets_sold: active_lottery.tickets_sold,
                prize_pool: active_lottery.prize_pool,
                timestamp: get_block_timestamp(),
            });
        }

        fn claim_prize(ref self: ContractState) {
            let active_lottery = self.active_lottery.read();
            let caller = get_caller_address();
            
            assert(active_lottery.lottery_status >= 2, Errors::LOTTERY_NOT_ENDED);
            
            let lottery_id = active_lottery.lottery_id;
            let winner_count = self.winner_count.read(lottery_id);
            
            let mut prize_found = false;
            let mut total_prize: u256 = 0;
            let mut i: u32 = 0;
            
            loop {
                if i >= winner_count {
                    break;
                }
                
                let winner = self.winners.read((lottery_id, i));
                if winner.address == caller && !winner.claimed {
                    prize_found = true;
                    total_prize += winner.prize_amount;
                    
                    self.winners.write(
                        (lottery_id, i),
                        Winner { 
                            address: winner.address, 
                            prize_amount: winner.prize_amount,
                            claimed: true 
                        }
                    );
                }
                
                i += 1;
            }
            
            assert(prize_found, Errors::NO_PRIZES_TO_CLAIM);
            assert(total_prize > 0, Errors::NO_PRIZES_TO_CLAIM);
            
            self.payment_token.read().transfer(caller, total_prize);
            
            self.emit(PrizeClaimed {
                lottery_id,
                winner: caller,
                prize_amount: total_prize,
            });
        }

        fn get_lottery_info(self: @ContractState) -> LotteryInfo {
            self.active_lottery.read()
        }

        fn get_ticket_holders(self: @ContractState) -> Array<ContractAddress> {
            let active_lottery = self.active_lottery.read();
            let mut holders = ArrayTrait::new();
            let mut i: u32 = 0;
            
            loop {
                if i >= active_lottery.tickets_sold {
                    break;
                }
                
                holders.append(self.ticket_holders.read(i));
                i += 1;
            }
            
            holders
        }

        fn get_winners(self: @ContractState) -> Array<Winner> {
            let active_lottery = self.active_lottery.read();
            let lottery_id = active_lottery.lottery_id;
            let winner_count = self.winner_count.read(lottery_id);
            let mut winners = ArrayTrait::new();
            let mut i: u32 = 0;
            
            loop {
                if i >= winner_count {
                    break;
                }
                
                winners.append(self.winners.read((lottery_id, i)));
                i += 1;
            }
            
            winners
        }

        fn get_user_tickets(self: @ContractState, user: ContractAddress) -> u32 {
            let active_lottery = self.active_lottery.read();
            self.user_tickets.read((active_lottery.lottery_id, user))
        }

        fn is_lottery_active(self: @ContractState) -> bool {
            let active_lottery = self.active_lottery.read();
            active_lottery.lottery_status == 1
        }
    }

    #[generate_trait]
    impl PrivateFunctions of PrivateFunctionsTrait {
        fn assert_only_owner(self: @ContractState) {
            let caller = get_caller_address();
            let owner = self.owner.read();
            assert(caller == owner, Errors::NOT_OWNER);
        }

        fn generate_random_value(self: @ContractState, seed: felt252) -> felt252 {
            let block_number = get_block_number();
            let block_timestamp = get_block_timestamp();
            let tx_info = get_tx_info().unbox();
            let tx_hash = tx_info.transaction_hash;
            
            let mut hash_state = PedersenTrait::new(0);
            hash_state = hash_state.update(seed);
            hash_state = hash_state.update(block_number.into());
            hash_state = hash_state.update(block_timestamp.into());
            hash_state = hash_state.update(tx_hash);
            
            hash_state.finalize()
        }

        fn select_random_ticket(
            self: @ContractState,
            total_tickets: u32,
            random_value: felt252,
            offset: u32
        ) -> u32 {
    
            let mut hash_state = PedersenTrait::new(0);
            hash_state = hash_state.update(random_value);
            hash_state = hash_state.update(offset.into());
            let new_random = hash_state.finalize();
            
            (new_random.try_into().unwrap() % total_tickets.into()).try_into().unwrap()
        }
    }
}
```
