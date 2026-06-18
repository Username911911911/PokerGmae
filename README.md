using System;
using System.Collections.Generic;
using System.Linq;

namespace TexasHoldemConsole
{
    // ==========================================
    // 1. קלפים וחפיסה (מתוך Card.cs)
    // ==========================================
    public enum Suit { Clubs, Diamonds, Hearts, Spades }
    public enum Rank { Two = 2, Three, Four, Five, Six, Seven, Eight, Nine, Ten, Jack, Queen, King, Ace }

    public class Card
    {
        public Suit CardSuit { get; }
        public Rank CardRank { get; }

        public Card(Suit suit, Rank rank)
        {
            CardSuit = suit;
            CardRank = rank;
        }

        public override string ToString()
        {
            return $"{CardRank} of {CardSuit}";
        }
    }

    public class Deck
    {
        private List<Card> cards;
        private Random random;

        public Deck()
        {
            cards = new List<Card>();
            random = new Random();
            Reset();
        }

        public void Reset()
        {
            cards.Clear();
            foreach (Suit suit in Enum.GetValues(typeof(Suit)))
            {
                foreach (Rank rank in Enum.GetValues(typeof(Rank)))
                {
                    cards.Add(new Card(suit, rank));
                }
            }
        }

        public void Shuffle()
        {
            int n = cards.Count;
            while (n > 1)
            {
                n--;
                int k = random.Next(n + 1);
                Card value = cards[k];
                cards[k] = cards[n];
                cards[n] = value;
            }
        }

        public Card DealCard()
        {
            if (cards.Count == 0) throw new InvalidOperationException("The deck is empty.");
            Card cardToDeal = cards[0];
            cards.RemoveAt(0);
            return cardToDeal;
        }
    }

    // ==========================================
    // 2. מחלקת שחקן (מתוך Player.cs)
    // ==========================================
    public class Player
    {
        public string Name { get; }
        public int Chips { get; set; }
        public List<Card> Hand { get; private set; }
        public bool IsFolded { get; set; }
        public int CurrentRoundBet { get; set; }
        public bool IsBot { get; }

        public Player(string name, int startingChips, bool isBot = false)
        {
            Name = name;
            Chips = startingChips;
            Hand = new List<Card>();
            IsFolded = false;
            CurrentRoundBet = 0;
            IsBot = isBot;
        }

        public void ReceiveCard(Card card)
        {
            Hand.Add(card);
        }

        public void ClearHand()
        {
            Hand.Clear();
            IsFolded = false;
            CurrentRoundBet = 0;
        }

        public void ShowHand()
        {
            if (Hand.Count >= 2)
            {
                Console.WriteLine($"{Name}'s cards: {Hand[0]} | {Hand[1]}");
            }
        }
    }

    // ==========================================
    // 3. חישוב ודירוג ידיים (מתוך HandValue.cs)
    // ==========================================
    public enum HandRank
    {
        HighCard,
        Pair,
        TwoPair,
        ThreeOfAKind,
        Straight,
        Flush,
        FullHouse,
        FourOfAKind,
        StraightFlush
    }

    public class HandValue
    {
        public HandRank Rank { get; set; }
        public int PrimaryStrength { get; set; }
    }

    public static class HandEvaluator
    {
        public static HandValue Evaluate5CardHand(List<Card> hand)
        {
            var orderedByRank = hand.OrderByDescending(c => (int)c.CardRank).ToList();
            
            bool isFlush = hand.Select(c => c.CardSuit).Distinct().Count() == 1;
            bool isStraight = IsHandStraight(orderedByRank);

            if (isFlush && isStraight) 
                return new HandValue { Rank = HandRank.StraightFlush, PrimaryStrength = (int)orderedByRank.First().CardRank };

            var groups = hand.GroupBy(c => c.CardRank)
                             .OrderByDescending(g => g.Count())
                             .ThenByDescending(g => g.Key)
                             .ToList();

            if (groups[0].Count() == 4)
                return new HandValue { Rank = HandRank.FourOfAKind, PrimaryStrength = (int)groups[0].Key };

            if (groups[0].Count() == 3 && groups[1].Count() >= 2)
                return new HandValue { Rank = HandRank.FullHouse, PrimaryStrength = (int)groups[0].Key };

            if (isFlush)
                return new HandValue { Rank = HandRank.Flush, PrimaryStrength = (int)orderedByRank.First().CardRank };

            if (isStraight)
                return new HandValue { Rank = HandRank.Straight, PrimaryStrength = (int)orderedByRank.First().CardRank };

            if (groups[0].Count() == 3)
                return new HandValue { Rank = HandRank.ThreeOfAKind, PrimaryStrength = (int)groups[0].Key };

            if (groups[0].Count() == 2 && groups[1].Count() == 2)
                return new HandValue { Rank = HandRank.TwoPair, PrimaryStrength = (int)groups[0].Key };

            if (groups[0].Count() == 2)
                return new HandValue { Rank = HandRank.Pair, PrimaryStrength = (int)groups[0].Key };

            return new HandValue { Rank = HandRank.HighCard, PrimaryStrength = (int)orderedByRank.First().CardRank };
        }

        private static bool IsHandStraight(List<Card> orderedCards)
        {
            for (int i = 0; i < orderedCards.Count - 1; i++)
            {
                if ((int)orderedCards[i].CardRank - (int)orderedCards[i + 1].CardRank != 1)
                    return false;
            }
            return true;
        }

        public static HandValue GetBest5CardHand(List<Card> holeCards, List<Card> communityCards)
        {
            List<Card> all7Cards = holeCards.Concat(communityCards).ToList();
            HandValue bestHand = new HandValue { Rank = HandRank.HighCard, PrimaryStrength = 0 };

            for (int i = 0; i < 7; i++)
            {
                for (int j = i + 1; j < 7; j++)
                {
                    var combo = all7Cards.Where((c, index) => index != i && index != j).ToList();
                    var currentEval = Evaluate5CardHand(combo);

                    if (currentEval.Rank > bestHand.Rank || 
                       (currentEval.Rank == bestHand.Rank && currentEval.PrimaryStrength > bestHand.PrimaryStrength))
                    {
                        bestHand = currentEval;
                    }
                }
            }
            return bestHand;
        }
    }

    // ==========================================
    // 4. מנוע המשחק (מתוך GameEngine.cs)
    // ==========================================
    public class GameEngine
    {
        private Deck deck;
        private List<Player> players;
        private List<Card> communityCards;
        private int pot;

        public GameEngine()
        {
            deck = new Deck();
            players = new List<Player>();
            communityCards = new List<Card>();
            pot = 0;
        }

        public void StartGame()
        {
            Console.WriteLine("--- Welcome to Texas Hold'em Poker ---");
            
            players.Add(new Player("Player 1 (You)", 1000, false));

            int botCount = 0;
            while (botCount < 1 || botCount > 5)
            {
                Console.Write("How many computer players (bots) do you want to play against? (1-5): ");
                if (!int.TryParse(Console.ReadLine(), out botCount) || botCount < 1 || botCount > 5)
                {
                    Console.WriteLine("Invalid input. Please enter a number between 1 and 5.");
                }
            }

            for (int i = 1; i <= botCount; i++)
            {
                players.Add(new Player($"CPU {i}", 1000, true));
            }

            Console.WriteLine($"\nGame starting with you and {botCount} computer players. Good luck!\n");

            bool keepPlaying = true;
            int roundNumber = 1;

            while (keepPlaying)
            {
                Console.WriteLine($"\n=================================");
                Console.WriteLine($"       STARTING ROUND {roundNumber}       ");
                Console.WriteLine($"=================================");
                
                PlayRound();

                if (players[0].Chips <= 0)
                {
                    Console.WriteLine("\n[GAME OVER] You ran out of chips! Better luck next time.");
                    break;
                }

                bool botsHaveChips = false;
                for (int i = 1; i < players.Count; i++)
                {
                    if (players[i].Chips > 0) botsHaveChips = true;
                }

                if (!botsHaveChips)
                {
                    Console.WriteLine("\n[CONGRATULATIONS!] You bankrupt all the bots! You are the poker champion!");
                    break;
                }

                Console.WriteLine("\n-> Press [ESC] to exit the game, or any other key to start the next round...");
                
                ConsoleKeyInfo keyInfo = Console.ReadKey(true); 
                if (keyInfo.Key == ConsoleKey.Escape)
                {
                    keepPlaying = false;
                    Console.WriteLine("\nExiting game... Here are the final standings:");
                    foreach (var p in players)
                    {
                        Console.WriteLine($"- {p.Name}: {p.Chips} chips");
                    }
                }
                else
                {
                    roundNumber++;
                }
            }
        }

        private void PlayRound()
        {
            deck.Reset();
            deck.Shuffle();
            communityCards.Clear();
            pot = 0;

            // איפוס מלא של מצב הידיים והפרישה מסיבוב קודם
            foreach (var player in players)
            {
                player.ClearHand();
            }

            int smallBlind = 10;
            int bigBlind = 20;

            int sbPaid = Math.Min(smallBlind, players[0].Chips);
            players[0].Chips -= sbPaid;
            players[0].CurrentRoundBet = sbPaid;
            pot += sbPaid;
            Console.WriteLine($"{players[0].Name} posts Small Blind: {sbPaid}");

            int bbPaid = Math.Min(bigBlind, players[1].Chips);
            players[1].Chips -= bbPaid;
            players[1].CurrentRoundBet = bbPaid;
            pot += bbPaid;
            Console.WriteLine($"{players[1].Name} posts Big Blind: {bbPaid}");

            for (int i = 0; i < 2; i++)
            {
                foreach (var player in players)
                {
                    if (player.Chips > 0 || player.CurrentRoundBet > 0)
                    {
                        player.ReceiveCard(deck.DealCard());
                    }
                    else
                    {
                        player.IsFolded = true; 
                    }
                }
            }

            // 1. Pre-Flop
            Console.WriteLine("\n--- Pre-Flop Betting Stage ---");
            ExecuteBettingRound(isPreFlop: true);
            if (CheckSinglePlayerLeft()) return;

            // 2. Flop
            Console.WriteLine("\n--- Flop ---");
            for (int i = 0; i < 3; i++) communityCards.Add(deck.DealCard());
            ExecuteBettingRound(isPreFlop: false);
            if (CheckSinglePlayerLeft()) return;

            // 3. Turn
            Console.WriteLine("\n--- Turn ---");
            communityCards.Add(deck.DealCard());
            ExecuteBettingRound(isPreFlop: false);
            if (CheckSinglePlayerLeft()) return;

            // 4. River
            Console.WriteLine("\n--- River ---");
            communityCards.Add(deck.DealCard());
            ExecuteBettingRound(isPreFlop: false);
            if (CheckSinglePlayerLeft()) return;
            
            // 5. Showdown
            ExecuteShowdown();
        }

        private void ExecuteBettingRound(bool isPreFlop)
        {
            if (!isPreFlop)
            {
                foreach (var p in players) p.CurrentRoundBet = 0;
            }

            int highestBetThisRound = isPreFlop ? 20 : 0; 
            bool bettingComplete = false;

            HashSet<string> putInAction = new HashSet<string>();

            while (!bettingComplete)
            {
                bettingComplete = true;

                foreach (var player in players)
                {
                    if (player.IsFolded || player.Chips == 0) continue;

                    int callAmount = highestBetThisRound - player.CurrentRoundBet;

                    if (!putInAction.Contains(player.Name) || callAmount > 0)
                    {
                        bettingComplete = false;
                        putInAction.Add(player.Name);

                        Console.WriteLine($"\n=================================");
                        if (!player.IsBot)
                        {
                            player.ShowHand();
                            PrintCommunityCards();
                        }
                        Console.WriteLine($"--- {player.Name}'s Turn ---");
                        Console.WriteLine($"Chips: {player.Chips} | Current Pot: {pot} | Highest Bet to Match: {highestBetThisRound}");
                        Console.WriteLine($"You already bet: {player.CurrentRoundBet} this round.");
                        Console.WriteLine($"=================================");

                        string choice = "2"; 

                        if (player.IsBot)
                        {
                            if (communityCards.Count < 5)
                            {
                                if (callAmount == 0)
                                {
                                    choice = new Random().Next(0, 10) < 2 ? "3" : "2";
                                }
                                else
                                {
                                    if (callAmount <= player.Chips * 0.5)
                                    {
                                        choice = new Random().Next(0, 100) < 15 ? "3" : "2";
                                    }
                                    else
                                    {
                                        choice = "1"; 
                                    }
                                }
                            }
                            else
                            {
                                HandValue currentBotHand = HandEvaluator.GetBest5CardHand(player.Hand, communityCards);
                                
                                if (callAmount == 0)
                                {
                                    if (currentBotHand.Rank > HandRank.HighCard && new Random().Next(0, 10) < 3) choice = "3";
                                    else choice = "2";
                                }
                                else 
                                {
                                    if (currentBotHand.Rank >= HandRank.TwoPair)
                                    {
                                        choice = new Random().Next(0, 10) < 4 ? "3" : "2";
                                    }
                                    else if (currentBotHand.Rank == HandRank.Pair)
                                    {
                                        choice = (callAmount <= player.Chips * 0.3) ? "2" : "1";
                                    }
                                    else
                                    {
                                        choice = (callAmount <= player.Chips * 0.05) ? "2" : "1";
                                    }
                                }
                            }
                            
                            System.Threading.Thread.Sleep(1200); 
                            Console.WriteLine($"{player.Name} (CPU) chooses option [{choice}]");
                        }
                        else
                        {
                            Console.WriteLine("Choose action: [1] Fold | [2] Check/Call | [3] Raise");
                            choice = Console.ReadLine();
                        }

                        if (choice == "1")
                        {
                            player.IsFolded = true;
                            Console.WriteLine($"{player.Name} Folds.");
                        }
                        else if (choice == "2")
                        {
                            if (callAmount == 0)
                            {
                                Console.WriteLine($"{player.Name} Checks.");
                            }
                            else
                            {
                                if (callAmount > player.Chips) callAmount = player.Chips;
                                player.Chips -= callAmount;
                                player.CurrentRoundBet += callAmount;
                                pot += callAmount;
                                Console.WriteLine($"{player.Name} Calls {callAmount}.");
                            }
                        }
                        else if (choice == "3")
                        {
                            int raiseAmount = 0;

                            if (player.IsBot)
                            {
                                raiseAmount = highestBetThisRound + 40; 
                            }
                            else
                            {
                                Console.Write($"Enter total amount to raise to (Must be higher than {highestBetThisRound}): ");
                                int.TryParse(Console.ReadLine(), out raiseAmount);
                            }

                            int additionalNeeded = raiseAmount - player.CurrentRoundBet;

                            if (raiseAmount > highestBetThisRound && additionalNeeded <= player.Chips)
                            {
                                player.Chips -= additionalNeeded;
                                player.CurrentRoundBet = raiseAmount;
                                pot += additionalNeeded;
                                highestBetThisRound = raiseAmount;
                                Console.WriteLine($"{player.Name} Raises to {raiseAmount}.");
                                bettingComplete = false; 
                            }
                            else
                            {
                                Console.WriteLine($"{player.Name} failed to raise. Automatically Checking/Calling instead.");
                                if (callAmount > player.Chips) callAmount = player.Chips;
                                player.Chips -= callAmount;
                                player.CurrentRoundBet += callAmount;
                                pot += callAmount;
                            }
                        }
                    }
                }

                int activePlayers = 0;
                foreach (var p in players) if (!p.IsFolded) activePlayers++;
                if (activePlayers <= 1) break; 
            }
            Console.WriteLine($"\nBetting round finished. Total Pot: {pot}");
        }

        private void ExecuteShowdown()
        {
            Console.WriteLine("\n--- Showdown Results ---");
            PrintCommunityCards();
            
            Player winner = null;
            HandValue bestHandValue = null;

            foreach (var player in players)
            {
                if (player.IsFolded) continue;

                HandValue playerBestHand = HandEvaluator.GetBest5CardHand(player.Hand, communityCards);
             
